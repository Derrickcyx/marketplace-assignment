<script>
import { mapState, mapGetters } from 'vuex'
import { get } from 'lodash'
import { matArchive, matDrafts, matMoreVert, matSearch, matUnarchive } from '@quasar/extras/material-icons'

import TransactionActions from 'src/components/TransactionActions'
import TransactionStatus from 'src/components/TransactionStatus'

import PageComponentMixin from 'src/mixins/pageComponent'

export default {
  components: {
    TransactionActions,
    TransactionStatus,
  },
  mixins: [PageComponentMixin],
  data() {
    return {
      filter: '',
      selectFilter: 'active',
      selected: [],
      columns: [
        // Simplified column definitions with dynamic translations
        { name: 'id', label: 'ID', align: 'left', field: row => row.id },
        { name: 'interlocutor', label: this.$t('user.name_label'), align: 'left', field: row => get(row, 'interlocutor.displayName', ''), sortable: true },
        { name: 'assetName', label: this.$t('asset.name_label'), align: 'left', field: row => get(row, 'asset.name', ''), sortable: true },
        { name: 'messages', label: 'Content', align: 'left', field: row => row.messages },
        { name: 'status', label: this.$t('transaction.status.label'), align: 'center', field: row => get(row, 'transaction.status', ''), sortable: true },
      ],
      rating: {
        dialogOpened: false,
        author: {},
        target: {},
        transaction: {},
        savedRatings: [],
        readonly: false
      },
      icons: {
        matArchive,
        matDrafts,
        matMoreVert,
        matUnarchive,
        matSearch
      }
    }
  },
  computed: {
    ...mapState(['style', 'auth']),
    ...mapGetters(['currentUser', 'conversations', 'ratingsActive']),
    // Combining and simplifying filtering logic
    filteredConversations() {
      const filterMap = {
        active: this.activeConversations,
        archive: this.archivedConversations,
        unread: this.unreadConversations,
      };
      return filterMap[this.selectFilter] || this.visibleConversations;
    },
    activeConversations() {
      return this.visibleConversations.filter(convo => !convo.archived);
    },
    archivedConversations() {
      return this.visibleConversations.filter(convo => convo.archived);
    },
    unreadConversations() {
      return this.visibleConversations.filter(convo => !convo.read);
    },
    visibleConversations() {
      return this.conversations.filter(convo => !convo.isEmpty);
    },
    selectFilterOptions() {
      return [
        { label: this.$t('pages.inbox.filter.active'), count: this.activeConversations.length, value: 'active' },
        { label: this.$t('pages.inbox.filter.unread'), count: this.unreadConversations.length, value: 'unread' },
        { label: this.$t('pages.inbox.filter.all'), count: this.visibleConversations.length, value: 'all' },
        { label: this.$t('pages.inbox.filter.archived'), count: this.archivedConversations.length, value: 'archive' },
      ];
    },
    selectFilterLabel() {
      return this.selectFilterOptions.find(option => option.value === this.selectFilter).label;
    },
  },
  methods: {
    afterAuth() {
      this.$store.dispatch('fetchMessages');
    },
    onSaveRatings() {
      this.$store.dispatch('fetchMessages', { forceRefreshAll: true });
    },
    archive(conversationId) {
      this.$store.dispatch('setConversationArchivedStatus', { conversationId, archived: true });
    },
    unarchive(conversationId) {
      this.$store.dispatch('setConversationArchivedStatus', { conversationId, archived: false });
    },
    markAsRead(conversationId) {
      this.$store.dispatch('markConversationAsRead', { conversationId });
    },
    searchConversations(rows, terms, cols) {
      const lowerTerms = terms ? terms.toLowerCase() : '';
      return rows.filter(row => {
        const rowMessages = row.messages.map(m => m.content.toLowerCase());
        const assetName = get(row, 'asset.name', '').toLowerCase();
        const interlocutorName = get(row, 'interlocutor.displayName', '').toLowerCase();

        return rowMessages.some(m => m.includes(lowerTerms)) || interlocutorName.includes(lowerTerms) || assetName.includes(lowerTerms);
      });
    }
  }
}
</script>
<template>
  <QPage class="stl-footer--bottom" padding>
    <div class="inbox stl-content-container stl-content-container--large margin-h-center">
      <AppContent tag="h1" class="text-h4 text-weight-medium" entry="pages" field="inbox.inbox_header" />

      <div class="row flex-center q-mb-md shadow-2">
        <QTable
          class="col-12"
          :title="$t('pages.inbox.inbox_header')"
          :data="filteredConversations"
          :columns="columns"
          :visible-columns="['interlocutor', 'assetName', 'messages', 'status']"
          :rows-per-page-options="[5, 10, 20, 50]"
          row-key="id"
          selection="multiple"
          :selected.sync="selected"
          :filter="filter"
          :filter-method="searchConversations"
          :hide-header="$q.screen.xs"
          grid
        >
          <template v-slot:top>
            <QSpace />
            <div v-if="$q.screen.gt.xs" class="col">
              <QBtnDropdown :label="selectFilterLabel" :rounded="style.roundedTheme" :flat="!style.colorfulTheme">
                <QList>
                  <QItem
                    v-for="option in selectFilterOptions"
                    :key="option.value"
                    v-close-popup
                    clickable
                    @click="selectFilter = option.value"
                  >
                    <QItemSection>
                      <QItemLabel class="row justify-between">
                        <div>{{ option.label }}</div>
                        <QBadge class="filter-count-badge q-ml-md" color="grey-3" text-color="default-color">{{ option.count || 0 }}</QBadge>
                      </QItemLabel>
                    </QItemSection>
                  </QItem>
                </QList>
              </QBtnDropdown>
            </div>
            <QInput
              v-model="filter"
              class="gt-xs"
              borderless
              dense
              debounce="300"
              :placeholder="$t('form.search.placeholder')"
            >
              <template #append>
                <QIcon :name="icons.matSearch" />
              </template>
            </QInput>
          </template>

          <!-- Continue with template content... -->

        </QTable>
      </div>
    </div>

    <AppFooter />
  </QPage>
</template>
