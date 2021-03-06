<template>
  <b-container fluid>
    <!-- User Interface controls -->
    <b-row>
      <b-col md="6" class="my-1">
        <b-form-group horizontal label="Filter" class="mb-0">
          <b-input-group>
            <b-form-input v-model="filter" placeholder="Type to Search" />
            <b-input-group-append>
              <b-btn :disabled="!filter" @click="filter = ''">Clear</b-btn>
            </b-input-group-append>
          </b-input-group>
        </b-form-group>
      </b-col>
      <b-col md="6" class="my-1">
        <b-form-group horizontal label="Sort" class="mb-0">
          <b-input-group>
            <b-form-select v-model="sortBy" :options="sortOptions">
              <option slot="first" :value="null">-- none --</option>
            </b-form-select>
            <b-form-select :disabled="!sortBy" v-model="sortDesc" slot="append">
              <option :value="false">Asc</option>
              <option :value="true">Desc</option>
            </b-form-select>
          </b-input-group>
        </b-form-group>
      </b-col>
      <b-col md="6" class="my-1">
        <b-form-group horizontal label="Sort direction" class="mb-0">
          <b-input-group>
            <b-form-select v-model="sortDirection" slot="append">
              <option value="asc">Asc</option>
              <option value="desc">Desc</option>
              <option value="last">Last</option>
            </b-form-select>
          </b-input-group>
        </b-form-group>
      </b-col>
      <b-col md="6" class="my-1">
        <b-form-group horizontal label="Per page" class="mb-0">
          <b-form-select :options="pageOptions" v-model="perPage" />
        </b-form-group>
      </b-col>
    </b-row>

    <!-- Hosts table element -->
    <b-table show-empty
             striped
             hover
             stacked="md"
             :items="items"
             :fields="fields"
             :current-page="currentPage"
             :per-page="perPage"
             :filter="filter"
             :sort-by.sync="sortBy"
             :sort-desc.sync="sortDesc"
             :sort-direction="sortDirection"
             @filtered="onFiltered"
    >
      <template slot="computer_name" slot-scope="row">{{row.value}}</template>
      <template slot="host_status" slot-scope="row">
        <b-badge href="#"
                 v-b-tooltip.hover
                 title="Show details"
                 @click.stop="showModalStatusDetails(row.item, $event.target)"
                 v-bind:variant="badgeHostStatus(row.value)"
        >
          {{row.value}}
        </b-badge>
      </template>
      <template slot="last_update_status" slot-scope="row">
        {{new Date(row.value['date']).toLocaleString()}}
      </template>
      <template slot="actions" slot-scope="row">
        <!-- We use @click.stop here to prevent a
            'row-clicked'event from also happening -->
        <b-button size="sm"
          @click.stop="info(row.item, row.index, $event.target)"
          class="mr-1"
        >
          Info modal
        </b-button>
        <b-button size="sm" @click.stop="row.toggleDetails">
          {{ row.detailsShowing ? 'Hide' : 'Show' }} Details
        </b-button>
      </template>
      <template slot="row-details" slot-scope="row">
        <b-card>
          <ul>
            <li v-for="(value, key) in row.item" :key="key">
              {{ key }}: {{ value}}
            </li>
          </ul>
        </b-card>
      </template>
    </b-table>

    <b-row>
      <b-col md="6" class="my-1">
        <b-pagination
          :total-rows="totalRows"
          :per-page="perPage"
          v-model="currentPage"
          class="my-0"
        />
      </b-col>
    </b-row>

    <!-- Info modal -->
    <b-modal id="modalInfo" @hide="resetModal" :title="modalInfo.title" ok-only>
      <pre>{{ modalInfo.content }}</pre>
    </b-modal>

    <!-- Status details modal -->
    <b-modal id="modalStatusDetails"
      @hide="resetModalStatusDetails"
      size="lg"
      :hide-footer="true"
    >
      <template slot="modal-title">
        Status details of <b>{{ this.host.computer_name }}</b>
      </template>
      <ModalStatusDetails
        v-if="host.last_update_status"
        :status="{last_update_status: host.last_update_status,
                  status: host.host_status}"
      />
    </b-modal>

  </b-container>
</template>

<script>
/**
* This component display host inventory (table) with
* their name, status and last update date
**/
import { mapState } from 'vuex'
import ModalStatusDetails from '@/components/modals/ModalStatusDetails'

export default {
  data () {
    return {
      fields: [
        { key: 'computer_name',
          label: 'Computer name',
          sortable: true,
          sortDirection: 'desc' },
        { key: 'host_status',
          label: 'Status',
          sortable: true,
          sortDirection: 'desc',
          'class': 'text-center' },
        { key: 'last_update_status', label: 'Last Update' },
        { key: 'actions', label: 'Actions' }
      ],
      currentPage: 1,
      perPage: 15,
      totalRows: 0,
      pageOptions: [ 15, 30, 45 ],
      sortBy: null,
      sortDesc: false,
      sortDirection: 'asc',
      filter: null,
      modalInfo: { title: '', content: '' },
      host: {}
    }
  },
  components: {
    ModalStatusDetails
  },
  mounted () {
    this.$store.dispatch('LOAD_WAPT_JSON')
  },
  computed: {
    sortOptions () {
      // Create an options list from our fields
      return this.fields
        .filter(f => f.sortable)
        .map(f => { return { text: f.label, value: f.key } })
    },
    ...mapState({
      items: state => state.Wapt.waptState
    })
  },
  methods: {
    showModalStatusDetails (item, badge) {
      this.host = item
      this.$root.$emit('bv::show::modal', 'modalStatusDetails', badge)
    },
    resetModalStatusDetails () {
      this.host = {}
    },
    info (item, index, button) {
      this.modalInfo.title = `Row index: ${index}`
      this.modalInfo.content = JSON.stringify(item, null, 2)
      this.$root.$emit('bv::show::modal', 'modalInfo', button)
    },
    resetModal () {
      this.modalInfo.title = ''
      this.modalInfo.content = ''
    },
    onFiltered (filteredItems) {
      // Trigger pagination to update the number of
      // buttons/pages due to filtering
      this.totalRows = filteredItems.length
      this.currentPage = 1
    },
    // Show different badge variant according
    // to host status
    badgeHostStatus (status) {
      switch (status) {
        case 'ERROR':
          return 'danger'
        case 'TO-UPGRADE':
          return 'warning'
        default:
          return 'success'
      }
    }
  }
}
</script>
