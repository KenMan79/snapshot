<script setup>
import { ref, computed, watch, onMounted, inject, watchEffect } from 'vue';
import { useRoute, useRouter } from 'vue-router';
import { useI18n } from '@/composables/useI18n';
import { getProposal, getResults, getProposalVotes } from '@/helpers/snapshot';
import { explorerUrl, getIpfsUrl } from '@/helpers/utils';
import { useModal } from '@/composables/useModal';
import { useTerms } from '@/composables/useTerms';
import { useProfiles } from '@/composables/useProfiles';
import { useApp } from '@/composables/useApp';
import { useSharing } from '@/composables/useSharing';
import { useWeb3 } from '@/composables/useWeb3';
import { useClient } from '@/composables/useClient';
import { useInfiniteLoader } from '@/composables/useInfiniteLoader';
import { useStore } from '@/composables/useStore';
import { useIntl } from '@/composables/useIntl';
import pending from '@/helpers/pending.json';

const props = defineProps({
  space: Object
});

const route = useRoute();
const router = useRouter();
const { domain } = useApp();
const { t, setPageTitle } = useI18n();
const { web3, web3Account } = useWeb3();
const { send, clientLoading } = useClient();
const { store } = useStore();
const notify = inject('notify');
const { formatRelativeTime, formatNumber } = useIntl();

const id = route.params.id;

const modalOpen = ref(false);
const selectedChoices = ref(null);
const loadingProposal = ref(true);
const loadedResults = ref(false);
const loadingResultsFailed = ref(false);
const loadedVotes = ref(false);
const proposal = ref(null);
const votes = ref([]);
const userVote = ref([]);
const results = ref({});
const modalStrategiesOpen = ref(false);

const isCreator = computed(() => proposal.value?.author === web3Account.value);
const isAdmin = computed(() => {
  const admins = (props.space.admins || []).map(admin => admin.toLowerCase());
  return admins.includes(web3Account.value?.toLowerCase());
});
const strategies = computed(
  // Needed for older proposal that are missing strategies
  () => proposal.value?.strategies ?? props.space?.strategies
);

const symbols = computed(() =>
  strategies.value.map(strategy => strategy.params.symbol || '')
);
const threeDotItems = computed(() => {
  const items = [{ text: t('duplicateProposal'), action: 'duplicate' }];
  if (isAdmin.value || isCreator.value)
    items.push({ text: t('deleteProposal'), action: 'delete' });
  return items;
});

const browserHasHistory = computed(() => window.history.state.back);

const { modalAccountOpen } = useModal();
const { modalTermsOpen, termsAccepted, acceptTerms } = useTerms(props.space.id);

function clickVote() {
  !web3.value.account
    ? (modalAccountOpen.value = true)
    : !termsAccepted.value && props.space.terms
    ? (modalTermsOpen.value = true)
    : (modalOpen.value = true);
}

async function loadProposal() {
  loadingProposal.value = true;
  proposal.value = await getProposal(id);
  // Redirect to proposal space.id if it doesn't match route key
  if (
    route.name === 'spaceProposal' &&
    props.space.id !== proposal.value.space.id
  ) {
    router.push({ name: 'error-404' });
  }
  loadingProposal.value = false;
  loadResults();
}

function formatProposalVotes(votes) {
  return votes.map(vote => {
    vote.balance = vote.vp;
    vote.scores = vote.vp_by_strategy;
    return vote;
  });
}

async function loadResults() {
  loadingResultsFailed.value = false;
  const spaceShowPending = pending;
  const showPending =
    proposal.value.scores_state === 'pending' &&
    spaceShowPending.includes(proposal.value.space.id);
  if (
    proposal.value.scores_state === 'invalid' &&
    proposal.value.state === 'closed'
  ) {
    loadingResultsFailed.value = true;
  } else if (proposal.value.scores_state === 'final' || showPending) {
    results.value = {
      resultsByVoteBalance: proposal.value.scores,
      resultsByStrategyScore: proposal.value.scores_by_strategy,
      sumOfResultsBalance: proposal.value.scores_total
    };
    loadedResults.value = true;
    loadingResultsFailed.value = false;
    const [userVotesRes, votesRes] = await Promise.all([
      await getProposalVotes(id, {
        first: 1,
        voter: web3Account.value
      }),
      await getProposalVotes(id, {
        first: 10
      })
    ]);
    userVote.value = formatProposalVotes(userVotesRes);
    votes.value = formatProposalVotes(votesRes);
    loadedVotes.value = true;
  } else {
    const votesTmp = await getProposalVotes(id);
    try {
      const resultsObj = await getResults(
        props.space,
        proposal.value,
        votesTmp
      );
      results.value = resultsObj.results;
      votes.value = resultsObj.votes;
      loadedResults.value = true;
      loadingResultsFailed.value = false;
      loadedVotes.value = true;
    } catch (e) {
      console.log(e);
      loadingResultsFailed.value = true;
    }
  }
}

const { loadBy, loadingMore, loadMore } = useInfiniteLoader(10);

async function loadMoreVotes() {
  const votesObj = await getProposalVotes(id, {
    first: loadBy,
    skip: votes.value.length
  });
  votes.value = votes.value.concat(formatProposalVotes(votesObj));
}

async function deleteProposal() {
  const result = await send(props.space, 'delete-proposal', {
    proposal: proposal.value
  });
  console.log('Result', result);
  if (result.id) {
    store.space.proposals = [];
    notify(['green', t('notify.proposalDeleted')]);
    router.push({ name: 'spaceProposals' });
  }
}

const {
  shareToTwitter,
  shareToFacebook,
  shareToClipboard,
  startShare,
  sharingIsSupported,
  sharingItems
} = useSharing();

function selectFromThreedotDropdown(e) {
  if (e === 'delete') deleteProposal();
  if (e === 'duplicate')
    router.push({
      name: 'spaceCreate',
      params: {
        key: proposal.value.space.id,
        sourceProposal: proposal.value.id
      }
    });
}

function selectFromShareDropdown(e) {
  if (e === 'shareToTwitter')
    shareToTwitter(props.space, proposal.value, window);
  else if (e === 'shareToFacebook')
    shareToFacebook(props.space, proposal.value, window);
  else if (e === 'shareToClipboard')
    shareToClipboard(props.space, proposal.value);
}

const { profiles, loadProfiles } = useProfiles();

watch(proposal, () => {
  loadProfiles([proposal.value.author]);
});

watch(web3Account, () => {
  const choice = route.query.choice;
  if (proposal.value && choice) {
    selectedChoices.value = parseInt(choice);
    clickVote();
  }
});

watchEffect(() => {
  if (props.space?.name && proposal.value?.title)
    setPageTitle('page.title.space.proposal', {
      proposal: proposal.value.title,
      space: props.space.name
    });
});

onMounted(async () => {
  await loadProposal();
  const choice = route.query.choice;
  if (proposal.value.type === 'approval') selectedChoices.value = [];
  if (web3Account.value && choice) {
    selectedChoices.value = parseInt(choice);
    clickVote();
  }
});

const showFullMarkdownBody = ref(false);

// Scroll to top of the page after clicking "Show less" button
watch(showFullMarkdownBody, () => {
  if (!showFullMarkdownBody.value) window.scrollTo(0, 0);
});

// Ref to the proposal body element
const markdownBody = ref(null);

// Detect if the proposal body is too long and should be shortened
const truncateMarkdownBody = computed(() => {
  const markdownBodyHeight = markdownBody.value?.clientHeight
    ? markdownBody.value.clientHeight
    : 0;
  return markdownBodyHeight > 400 ? true : false;
});
</script>

<template>
  <TheLayout v-bind="$attrs">
    <template #content-left>
      <div class="px-3 md:px-0 mb-3">
        <a
          class="text-skin-text"
          @click="
            browserHasHistory?.includes('timeline')
              ? $router.go(-1)
              : $router.push(
                  domain ? { path: '/' } : { name: 'spaceProposals' }
                )
          "
        >
          <BaseIcon name="back" size="22" class="align-middle" />
          {{ $t('back') }}
        </a>
      </div>
      <div class="px-3 md:px-0">
        <template v-if="proposal">
          <h1
            v-text="proposal.title"
            class="mb-3 break-words text-xl leading-8 sm:text-2xl"
          />

          <div class="flex flex-col sm:flex-row sm:space-x-1 mb-4">
            <div class="flex items-center mb-1 sm:mb-0">
              <LabelProposalState :state="proposal.state" class="mr-2" />
              <router-link
                class="text-skin-text group"
                :to="{
                  name: 'spaceProposals',
                  params: { key: space.id }
                }"
              >
                <div class="flex items-center">
                  <AvatarSpace :space="space" size="28" />
                  <span
                    class="ml-2 group-hover:text-skin-link"
                    v-text="space.name"
                  />
                </div>
              </router-link>
            </div>
            <div class="flex grow items-center space-x-1">
              <span v-text="$t('proposalBy')" />
              <AvatarUser
                :address="proposal.author"
                :profile="profiles[proposal.author]"
                :space="space"
                :proposal="proposal"
                only-username
              />
              <BaseBadge :address="proposal.author" :members="space.members" />
              <ShareButton
                v-if="sharingIsSupported"
                @click="startShare(space, proposal)"
              />
              <BaseDropdown
                v-else
                class="pl-3 !ml-auto"
                @select="selectFromShareDropdown"
                :items="sharingItems"
              >
                <template v-slot:button>
                  <ShareButton />
                </template>
                <template v-slot:item="{ item }">
                  <BaseIcon
                    v-if="item.icon"
                    :name="item.icon"
                    size="21"
                    class="align-middle mr-2 !leading-[0]"
                  />
                  {{ item.text }}
                </template>
              </BaseDropdown>
              <BaseDropdown
                class="md:ml-2"
                @select="selectFromThreedotDropdown"
                :items="threeDotItems"
              >
                <template v-slot:button>
                  <div class="pl-1">
                    <LoadingSpinner v-if="clientLoading" />
                    <BaseIcon
                      v-else
                      name="threedots"
                      size="25"
                      class="hover:text-skin-link"
                    />
                  </div>
                </template>
              </BaseDropdown>
            </div>
          </div>

          <div class="relative">
            <div
              v-if="!showFullMarkdownBody && truncateMarkdownBody"
              class="absolute w-full h-[80px] bottom-0 bg-gradient-to-t from-skin-bg"
            />
            <div
              v-if="truncateMarkdownBody"
              class="absolute w-full flex justify-center"
              :class="{
                '-bottom-[64px]': showFullMarkdownBody,
                '-bottom-[14px]': !showFullMarkdownBody
              }"
            >
              <BaseButton
                @click="showFullMarkdownBody = !showFullMarkdownBody"
                class="!bg-skin-bg"
              >
                {{
                  showFullMarkdownBody
                    ? $t('proposals.showLess')
                    : $t('proposals.showMore')
                }}
              </BaseButton>
            </div>
            <div
              class="overflow-hidden"
              :class="{
                'h-[360px]': !showFullMarkdownBody && truncateMarkdownBody,
                'mb-[92px]': showFullMarkdownBody,
                'mb-[56px]': !showFullMarkdownBody
              }"
            >
              <div ref="markdownBody">
                <BaseMarkdown :body="proposal.body" />
              </div>
            </div>
          </div>
        </template>
        <LoadingPage v-else />
      </div>
      <div class="space-y-4 py-4">
        <BaseLink
          v-if="proposal?.discussion"
          :link="getIpfsUrl(proposal.discussion)"
          hide-external-icon
        >
          <BaseBlock class="cursor-pointer hover:border-skin-text">
            <div class="text-skin-link flex items-center justify-between">
              <div class="flex items-center">
                <BaseIcon name="receipt-outlined" size="22" />
                <span class="ml-3 text-md">
                  {{ $t('discussion') }}
                </span>
              </div>
              <BaseIcon name="external-link" size="20" />
            </div>
          </BaseBlock>
        </BaseLink>
        <BlockCastVote
          v-if="proposal?.state === 'active'"
          :proposal="proposal"
          v-model="selectedChoices"
          @open="modalOpen = true"
          @clickVote="clickVote"
        />
        <BlockVotes
          @loadVotes="loadMore(loadMoreVotes)"
          v-if="proposal && !loadingResultsFailed"
          :loaded="loadedVotes"
          :space="space"
          :proposal="proposal"
          :votes="votes"
          :strategies="strategies"
          :userVote="userVote"
          :loadingMore="loadingMore"
        />
        <PluginProposal
          v-if="proposal?.plugins && loadedResults"
          :id="id"
          :space="space"
          :proposal="proposal"
          :results="results"
          :loadedResults="loadedResults"
          :votes="votes"
          :strategies="strategies"
        />
      </div>
    </template>
    <template #sidebar-right v-if="proposal">
      <div class="space-y-4 mt-4 lg:mt-0">
        <BaseBlock :title="$t('information')">
          <div class="space-y-1">
            <div>
              <b>{{ $t('strategies') }}</b>
              <span
                @click="modalStrategiesOpen = true"
                class="float-right text-skin-link a"
              >
                <span v-for="(symbol, symbolIndex) of symbols" :key="symbol">
                  <span
                    v-tippy="{
                      content: symbol
                    }"
                  >
                    <AvatarSpace :space="space" :symbolIndex="symbolIndex" />
                  </span>
                  <span
                    v-show="symbolIndex !== symbols.length - 1"
                    class="ml-1"
                  />
                </span>
              </span>
            </div>

            <div>
              <b>IPFS</b>
              <BaseLink :link="getIpfsUrl(proposal.ipfs)" class="float-right">
                #{{ proposal.ipfs.slice(0, 7) }}
              </BaseLink>
            </div>
            <div>
              <b>{{ $t('proposal.votingSystem') }}</b>
              <span class="float-right text-skin-link">
                {{ $t(`voting.${proposal.type}`) }}
              </span>
            </div>
            <div>
              <b>{{ $t('proposal.startDate') }}</b>
              <span
                v-text="$d(proposal.start * 1e3, 'short', 'en-US')"
                v-tippy="{
                  content: formatRelativeTime(proposal.start)
                }"
                class="float-right text-skin-link"
              />
            </div>
            <div>
              <b>{{ $t('proposal.endDate') }}</b>
              <span
                v-text="$d(proposal.end * 1e3, 'short', 'en-US')"
                v-tippy="{
                  content: formatRelativeTime(proposal.end)
                }"
                class="text-skin-link float-right"
              />
            </div>
            <div>
              <b>{{ $t('snapshot') }}</b>
              <BaseLink
                :link="
                  explorerUrl(proposal.network, proposal.snapshot, 'block')
                "
                class="float-right"
              >
                {{ formatNumber(proposal.snapshot) }}
              </BaseLink>
            </div>
          </div>
        </BaseBlock>
        <BlockResultsError
          v-if="loadingResultsFailed"
          :isAdmin="isAdmin"
          :proposalId="proposal.id"
          :proposalState="proposal.scores_state"
          @retry="loadProposal()"
        />
        <BlockResults
          v-else
          :id="id"
          :loaded="loadedResults"
          :space="space"
          :proposal="proposal"
          :results="results"
          :votes="votes"
          :strategies="strategies"
        />
        <PluginProposalSidebar
          v-if="proposal.plugins && loadedResults"
          :id="id"
          :space="space"
          :proposal="proposal"
          :results="results"
          :loadedResults="loadedResults"
          :votes="votes"
          :strategies="strategies"
        />
      </div>
    </template>
  </TheLayout>
  <teleport to="#modal" v-if="proposal">
    <ModalConfirm
      :open="modalOpen"
      @close="modalOpen = false"
      @reload="loadProposal()"
      :space="space"
      :proposal="proposal"
      :id="id"
      :selectedChoices="selectedChoices"
      :snapshot="proposal.snapshot"
      :strategies="strategies"
    />
    <ModalStrategies
      :open="modalStrategiesOpen"
      @close="modalStrategiesOpen = false"
      :proposal="proposal"
      :strategies="strategies"
    />
    <ModalTerms
      :open="modalTermsOpen"
      :space="space"
      @close="modalTermsOpen = false"
      @accept="acceptTerms(), (modalOpen = true)"
    />
  </teleport>
</template>
