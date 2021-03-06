<template>
  <b-card no-body>
    <b-card-header>
      <div class="float-right"><toggle-button v-model="advancedMode">{{ $t('Advanced') }}</toggle-button></div>
      <h4 class="mb-0" v-t="'Search Nodes'"></h4>
    </b-card-header>
    <pf-search :fields="fields" :store="$store" :advanced-mode="advancedMode" :condition="condition"
      @submit-search="onSearch" @reset-search="onReset" @import-search="onImport"></pf-search>
    <div class="card-body">
      <b-row align-h="between" align-v="center">
        <b-col cols="auto" class="mr-auto">
          <b-dropdown size="sm" variant="link" :disabled="isLoading || checkedRows.length === 0" no-caret>
            <template slot="button-content">
              <icon name="cogs" v-b-tooltip.hover.right :title="$t('Actions')"></icon>
            </template>
            <b-dropdown-item @click="applyBulkClearViolation()">
              <icon class="position-absolute mt-1" name="ban"></icon>
              <span class="ml-4">{{ $t('Clear Violation') }}</span>
            </b-dropdown-item>
            <b-dropdown-item @click="applyBulkRegister()">
              <icon class="position-absolute mt-1" name="plus-circle"></icon>
              <span class="ml-4">{{ $t('Register') }}</span>
            </b-dropdown-item>
            <b-dropdown-item @click="applyBulkDeregister()">
              <icon class="position-absolute mt-1" name="minus-circle"></icon>
              <span class="ml-4">{{ $t('Deregister') }}</span>
            </b-dropdown-item>
            <b-dropdown-item @click="applyBulkReevaluateAccess()">
              <icon class="position-absolute mt-1" name="sync"></icon>
              <span class="ml-4">{{ $t('Revaluate Access') }}</span>
            </b-dropdown-item>
            <b-dropdown-item @click="applyBulkRestartSwitchport()">
              <icon class="position-absolute mt-1" name="retweet"></icon>
              <span class="ml-4">{{ $t('Restart Switchport') }}</span>
            </b-dropdown-item>
            <b-dropdown-divider></b-dropdown-divider>
            <b-dropdown-header>{{ $t('Apply Role') }}</b-dropdown-header>
            <b-dropdown-item v-for="role in roles" :key="role.category_id" @click="applyBulkRole(role)" v-b-tooltip.hover.left :title="role.notes">
              <span>{{role.name}}</span>
            </b-dropdown-item>
            <b-dropdown-item @click="applyBulkRole({category_id: null})" v-b-tooltip.hover.left :title="$t('Clear Role')">
              <icon class="position-absolute mt-1" name="trash-alt"></icon>
              <span class="ml-4"><em>{{ $t('None') }}</em></span>
            </b-dropdown-item>
            <b-dropdown-divider></b-dropdown-divider>
            <b-dropdown-header>{{ $t('Apply Bypass Role') }}</b-dropdown-header>
            <b-dropdown-item v-for="role in roles" :key="role.category_id" @click="applyBulkBypassRole(role)" v-b-tooltip.hover.left :title="role.notes">
              <span>{{role.name}}</span>
            </b-dropdown-item>
            <b-dropdown-item @click="applyBulkBypassRole({category_id: null})" v-b-tooltip.hover.left :title="$t('Clear Bypass Role')">
              <icon class="position-absolute mt-1" name="trash-alt"></icon>
              <span class="ml-4"><em>{{ $t('None') }}</em></span>
            </b-dropdown-item>
            <b-dropdown-divider></b-dropdown-divider>
            <b-dropdown-header>{{ $t('Apply Violation') }}</b-dropdown-header>
            <b-dropdown-item v-for="violation in violations" v-if="violation.enabled ==='Y'" :key="violation.id" @click="applyBulkViolation(violation)" v-b-tooltip.hover.left :title="violation.id">
              <span>{{violation.desc}}</span>
            </b-dropdown-item>
          </b-dropdown>
          <b-dropdown size="sm" variant="link" boundary="viewport" :disabled="isLoading" no-caret>
            <template slot="button-content">
              <icon name="columns" v-b-tooltip.hover.right :title="$t('Visible Columns')"></icon>
            </template>
            <b-dropdown-item v-for="column in columns" :key="column.key" @click="toggleColumn(column)" :disabled="column.locked">
              <icon class="position-absolute mt-1" name="thumbtack" v-show="column.visible" v-if="column.locked"></icon>
              <icon class="position-absolute mt-1" name="check" v-show="column.visible" v-else></icon>
              <span class="ml-4">{{column.label}}</span>
            </b-dropdown-item>
          </b-dropdown>
        </b-col>
        <b-col cols="auto">
          <b-container fluid>
            <b-row align-v="center">
              <b-form inline class="mb-0">
                <b-form-select class="mb-3 mr-3" size="sm" v-model="pageSizeLimit" :options="[10,25,50,100]" :disabled="isLoading"
                  @input="onPageSizeChange" />
              </b-form>
              <b-pagination align="right" v-model="requestPage" :per-page="pageSizeLimit" :total-rows="totalRows" :disabled="isLoading"
                @input="onPageChange" />
            </b-row>
          </b-container>
        </b-col>
      </b-row>
      <b-table stacked="sm" :items="items" :fields="visibleColumns" :sort-by="sortBy" :sort-desc="sortDesc"
        @sort-changed="onSortingChanged" @row-clicked="onRowClick" @head-clicked="clearChecked" hover no-local-sorting v-model="tableValues">
        <template slot="HEAD_actions" slot-scope="head">
          <input type="checkbox" id="checkallnone" v-model="checkedAll" @change="onCheckedAllChange" @click.stop>
          <b-tooltip target="checkallnone" placement="right" v-if="checkedRows.length === tableValues.length">{{$t('Select None [ALT+N]')}}</b-tooltip>
          <b-tooltip target="checkallnone" placement="right" v-else>{{$t('Select All [ALT+A]')}}</b-tooltip>
        </template>
        <template slot="actions" slot-scope="data">
          <input type="checkbox" :id="data.value" :value="data.item" v-model="checkedRows" @click.stop="toggleCheckbox($event, data.index)">
          <icon name="exclamation-triangle" class="ml-1" v-if="tableValues[data.index]._message" v-b-tooltip.hover.right :title="tableValues[data.index]._message"></icon>
        </template>
        <template slot="status" slot-scope="data">
          <b-badge pill variant="success" v-if="data.value === 'reg'">{{ $t('registered') }}</b-badge>
          <b-badge pill variant="light" v-else>{{ $t('unregistered') }}</b-badge>
        </template>
        <template slot="device_score" slot-scope="data">
          <pf-fingerbank-score :score="data.value"></pf-fingerbank-score>
        </template>
      </b-table>
    </div>
  </b-card>
</template>

<script>
import { pfSearchConditionType as conditionType } from '@/globals/pfSearch'
import pfFingerbankScore from '@/components/pfFingerbankScore'
import pfSearch from '@/components/pfSearch'
import ToggleButton from '@/components/ToggleButton'

export default {
  name: 'NodesSearch',
  components: {
    'pf-fingerbank-score': pfFingerbankScore,
    'pf-search': pfSearch,
    'toggle-button': ToggleButton
  },
  props: {
    namedSearch: String,
    tableValues: {
      type: Array,
      default: []
    },
    checkedRows: {
      type: Array,
      default: []
    },
    checkedAll: {
      type: Boolean,
      default: false
    },
    lastIndex: {
      type: Number,
      default: null
    },
    query: {
      type: String,
      default: null
    }
  },
  data () {
    return {
      advancedMode: false,
      /**
       *  Fields on which a search can be defined.
       *  The names must match the database schema.
       *  The keys must conform to the format of the b-form-select's options property.
       */
      fields: [ // keys match with b-form-select
        {
          value: 'mac',
          text: this.$i18n.t('MAC Address [✓]'),
          types: [conditionType.SUBSTRING]
        },
        {
          value: 'bypass_role_id',
          text: this.$i18n.t('Bypass Role [✓]'),
          types: [conditionType.ROLE, conditionType.SUBSTRING]
        },
        {
          value: 'bypass_vlan',
          text: this.$i18n.t('Bypass VLAN [?]'),
          types: [conditionType.SUBSTRING]
        },
        {
          value: 'computername',
          text: this.$i18n.t('Computer Name [✓]'),
          types: [conditionType.SUBSTRING]
        },
        {
          value: 'locationlog.connection_type',
          text: this.$i18n.t('Connection Type [?]'),
          types: [conditionType.CONNECTION_TYPE]
        },
        {
          value: 'device_class',
          text: this.$i18n.t('Device Class [✓]'),
          types: [conditionType.SUBSTRING]
        },
        {
          value: 'device_manufacturer',
          text: this.$i18n.t('Device Manufacturer [?]'),
          types: [conditionType.SUBSTRING]
        },
        {
          value: 'device_type',
          text: this.$i18n.t('Device Type [✓]'),
          types: [conditionType.SUBSTRING]
        },
        {
          value: 'ip4log.ip',
          text: this.$i18n.t('IPv4 Address [✓]'),
          types: [conditionType.SUBSTRING]
        },
        {
          value: 'ip6log.ip',
          text: this.$i18n.t('IPv6 Address [✓]'),
          types: [conditionType.SUBSTRING]
        },
        {
          value: 'machine_account',
          text: this.$i18n.t('Machine Account [✓]'),
          types: [conditionType.SUBSTRING]
        },
        {
          value: 'notes',
          text: this.$i18n.t('Notes [✓]'),
          types: [conditionType.SUBSTRING]
        },
        {
          value: 'online',
          text: this.$i18n.t('Online Status [x]'),
          types: [conditionType.ONLINE]
        },
        {
          value: 'pid',
          text: this.$i18n.t('Owner [✓]'),
          types: [conditionType.SUBSTRING]
        },
        {
          value: 'category_id',
          text: this.$i18n.t('Role [✓]'),
          types: [conditionType.ROLE, conditionType.SUBSTRING]
        },
        {
          value: 'locationlog.switch',
          text: this.$i18n.t('Source Switch Identifier [✓]'),
          types: [conditionType.SUBSTRING]
        },
        {
          value: 'locationlog.switch_ip',
          text: this.$i18n.t('Source Switch IP [✓]'),
          types: [conditionType.SUBSTRING]
        },
        {
          value: 'locationlog.switch_mac',
          text: this.$i18n.t('Source Switch MAC [✓]'),
          types: [conditionType.SUBSTRING]
        },
        {
          value: 'locationlog.ssid',
          text: this.$i18n.t('SSID [✓]'),
          types: [conditionType.SUBSTRING]
        },
        {
          value: 'user_agent',
          text: this.$i18n.t('User Agent [✓]'),
          types: [conditionType.SUBSTRING]
        },
        {
          value: 'violation',
          text: this.$i18n.t('Violation Name [?]'),
          types: [conditionType.SUBSTRING]
        },
        {
          value: 'violation_status',
          text: this.$i18n.t('Violation Status [?]'),
          types: [conditionType.SUBSTRING]
        },
        {
          value: 'voip',
          text: this.$i18n.t('VoIP [✓]'),
          types: [conditionType.VOIP]
        }
      ],
      /**
       * The columns that can be displayed in the results table.
       */
      columns: [
        {
          key: 'actions',
          label: this.$i18n.t('Actions'),
          sortable: false,
          visible: true,
          locked: true,
          formatter: (value, key, item) => {
            return item.mac
          }
        },
        {
          key: 'status',
          label: this.$i18n.t('Status'),
          sortable: true,
          visible: true
        },
        {
          key: 'online',
          label: this.$i18n.t('Online/Offline'),
          sortable: true,
          visible: true
        },
        {
          key: 'mac',
          label: this.$i18n.t('MAC Address'),
          sortable: true,
          visible: true
        },
        {
          key: 'detect_date',
          label: this.$i18n.t('Detected Date'),
          sortable: true,
          visible: false
        },
        {
          key: 'regdate',
          label: this.$i18n.t('Registration Date'),
          sortable: true,
          visible: false
        },
        {
          key: 'unregdate',
          label: this.$i18n.t('Unregistration Date'),
          sortable: true,
          visible: false
        },
        {
          key: 'computername',
          label: this.$i18n.t('Computer Name'),
          sortable: true,
          visible: true
        },
        {
          key: 'pid',
          label: this.$i18n.t('Owner'),
          sortable: true,
          visible: true
        },
        {
          key: 'ip4log.ip',
          label: this.$i18n.t('IPv4 Address'),
          sortable: true,
          visible: true
        },
        {
          key: 'ip6log.ip',
          label: this.$i18n.t('IPv6 Address'),
          sortable: true,
          visible: true
        },
        {
          key: 'tenant_id',
          label: this.$i18n.t('Tenant'),
          sortable: true,
          visible: true
        },
        {
          key: 'device_class',
          label: this.$i18n.t('Device Class'),
          sortable: true,
          visible: true
        },
        {
          key: 'device_manufacturer',
          label: this.$i18n.t('Device Manufacturer'),
          sortable: true,
          visible: false
        },
        {
          key: 'device_score',
          label: this.$i18n.t('Device Score'),
          sortable: true,
          visible: false
        },
        {
          key: 'device_type',
          label: this.$i18n.t('Device Type'),
          sortable: true,
          visible: true
        },
        {
          key: 'device_version',
          label: this.$i18n.t('Device Version'),
          sortable: true,
          visible: false
        },
        {
          key: 'dhcp6_enterprise',
          label: this.$i18n.t('DHCPv6 Enterprise'),
          sortable: true,
          visible: false
        },
        {
          key: 'dhcp6_fingerprint',
          label: this.$i18n.t('DHCPv6 Fingerprint'),
          sortable: true,
          visible: false
        },
        {
          key: 'dhcp_fingerprint',
          label: this.$i18n.t('DHCP Fingerprint'),
          sortable: true,
          visible: false
        },
        {
          key: 'category_id',
          label: this.$i18n.t('Role'),
          sortable: true,
          visible: true,
          formatter: (value, key, item) => {
            return this.$store.state.config.roles.filter(role => role.category_id === item.category_id).map(role => role.name)
          }
        },
        {
          key: 'locationlog.connection_type',
          label: this.$i18n.t('Connection Type'),
          sortable: true,
          visible: false
        },
        {
          key: 'locationlog.session_id',
          label: this.$i18n.t('Session ID'),
          sortable: true,
          visible: false
        },
        {
          key: 'locationlog.switch',
          label: this.$i18n.t('Switch Identifier'),
          sortable: true,
          visible: false
        },
        {
          key: 'locationlog.switch_ip',
          label: this.$i18n.t('Switch IP Address'),
          sortable: true,
          visible: false
        },
        {
          key: 'locationlog.switch_mac',
          label: this.$i18n.t('Switch MAC Address'),
          sortable: true,
          visible: false
        },
        {
          key: 'locationlog.ssid',
          label: this.$i18n.t('SSID'),
          sortable: true,
          visible: false
        },
        {
          key: 'locationlog.vlan',
          label: this.$i18n.t('VLAN'),
          sortable: true,
          visible: false
        },
        {
          key: 'bypass_vlan',
          label: this.$i18n.t('Bypass VLAN'),
          sortable: true,
          visible: false
        },
        {
          key: 'bypass_role_id',
          label: this.$i18n.t('Bypass Role'),
          sortable: true,
          visible: false,
          formatter: (value, key, item) => {
            return this.$store.state.config.roles.filter(role => role.category_id === item.bypass_role_id).map(role => role.name)
          }
        },
        {
          key: 'notes',
          label: this.$i18n.t('Notes'),
          sortable: true,
          visible: false
        },
        {
          key: 'voip',
          label: this.$i18n.t('VoIP'),
          sortable: true,
          visible: false
        },
        {
          key: 'last_arp',
          label: this.$i18n.t('Last ARP'),
          sortable: true,
          visible: false
        },
        {
          key: 'last_dhcp',
          label: this.$i18n.t('Last DHCP'),
          sortable: true,
          visible: false
        },
        {
          key: 'machine_account',
          label: this.$i18n.t('Machine Account'),
          sortable: true,
          visible: false
        },
        {
          key: 'time_balance',
          label: this.$i18n.t('Time Balance'),
          sortable: true,
          visible: false
        },
        {
          key: 'user_agent',
          label: this.$i18n.t('User Agent'),
          sortable: true,
          visible: false
        }
      ],
      condition: null,
      requestPage: 1,
      currentPage: 1,
      pageSizeLimit: 10
    }
  },
  computed: {
    isLoading () {
      return this.$store.getters['$_nodes/isLoadingResults']
    },
    sortBy () {
      return this.$store.state.$_nodes.searchSortBy
    },
    sortDesc () {
      return this.$store.state.$_nodes.searchSortDesc
    },
    visibleColumns () {
      return this.columns.filter(column => column.visible)
    },
    searchFields () {
      return this.visibleColumns.filter(column => !column.locked).map(column => column.key)
    },
    items () {
      return this.$store.state.$_nodes.items
    },
    totalRows () {
      return this.$store.state.$_nodes.searchMaxPageNumber * this.pageSizeLimit
    },
    roles () {
      return this.$store.state.config.roles
    },
    violations () {
      return this.$store.getters['config/sortedViolations']
    }
  },
  methods: {
    onSearch (condition) {
      let _this = this
      this.requestPage = 1 // reset to the first page
      this.clearChecked()
      this.$store.dispatch('$_nodes/setSearchQuery', condition)
      this.$store.dispatch('$_nodes/search', this.requestPage).then(() => {
        _this.currentPage = _this.requestPage
        _this.condition = condition
      }).catch(() => {
        _this.requestPage = _this.currentPage
      })
    },
    onReset () {
      this.$router.push({ name: 'nodes' })
    },
    onImport (condition) {
      this.$router.push({ name: 'nodes', query: { query: JSON.stringify(condition) } })
    },
    onPageSizeChange () {
      this.requestPage = 1 // reset to the first page
      this.$store.dispatch('$_nodes/setSearchPageSize', this.pageSizeLimit)
      this.$store.dispatch('$_nodes/search', this.requestPage)
    },
    onPageChange () {
      let _this = this
      this.$store.dispatch('$_nodes/search', this.requestPage).then(() => {
        _this.currentPage = _this.requestPage
      }).catch(() => {
        _this.requestPage = _this.currentPage
      })
    },
    onSortingChanged (params) {
      this.requestPage = 1 // reset to the first page
      this.$store.dispatch('$_nodes/setSearchSorting', params)
      this.$store.dispatch('$_nodes/search', this.requestPage)
    },
    toggleColumn (column) {
      column.visible = !column.visible
      this.$store.dispatch('$_nodes/setVisibleColumns', this.columns.filter(column => column.visible).map(column => column.key))
      this.$store.dispatch('$_nodes/setSearchFields', this.searchFields)
      if (column.visible) {
        this.$store.dispatch('$_nodes/search', this.requestPage)
      }
    },
    onRowClick (item, index) {
      this.$router.push({ name: 'node', params: { mac: item.mac } })
    },
    onCheckedAllChange (item) {
      this.checkedRows = this.checkedAll ? this.tableValues : []
    },
    clearChecked () {
      this.checkedAll = false
      const _this = this
      this.checkedRows.forEach(function (item, index, items) {
        _this.$store.commit('$_nodes/ITEM_VARIANT', {mac: item.mac, variant: ''})
        _this.$store.commit('$_nodes/ITEM_MESSAGE', {mac: item.mac, message: ''})
      })
      this.checkedRows = []
      this.lastIndex = null
    },
    toggleCheckbox (event, index) {
      // support SHIFT+CLICK
      const lastIndex = this.lastIndex
      this.lastIndex = index
      if (lastIndex === null || index === lastIndex || !event.shiftKey) return
      const subset = this.items.slice(
        Math.min(index, lastIndex),
        Math.max(index, lastIndex) + 1
      )
      if (event.currentTarget.checked) {
        this.checkedRows.push(...subset)
        // remove duplicates
        this.checkedRows = this.checkedRows.reduce((x, y) => x.includes(y) ? x : [...x, y], [])
      } else {
        this.checkedRows = this.checkedRows.reduce((x, y) => subset.includes(y) ? x : [...x, y], [])
      }
    },
    applyBulkClearViolation () {
      const _this = this
      const macs = this.checkedRows.map(item => item.mac)
      if (macs.length > 0) {
        _this.$store.dispatch('$_nodes/clearViolationBulkNodes', {items: macs}).then(response => {
          response.items.forEach(function (item, index, items) {
            _this.$store.commit('$_nodes/ITEM_VARIANT', {mac: item.mac, status: item.status})
            _this.$store.commit('$_nodes/ITEM_MESSAGE', {mac: item.mac, message: item.message})
          })
        }).catch(() => {
          macs.forEach(function (mac, index) {
            _this.$store.commit('$_nodes/ITEM_VARIANT', {mac: mac, variant: 'danger'})
          })
        })
      }
    },
    applyBulkRegister () {
      const _this = this
      const macs = this.checkedRows.map(item => item.mac)
      if (macs.length > 0) {
        _this.$store.dispatch('$_nodes/registerBulkNodes', {items: macs}).then(response => {
          response.items.forEach(function (item, index, items) {
            _this.$store.commit('$_nodes/ITEM_VARIANT', {mac: item.mac, status: item.status})
            _this.$store.commit('$_nodes/ITEM_MESSAGE', {mac: item.mac, message: item.message})
          })
        }).catch(() => {
          macs.forEach(function (mac, index) {
            _this.$store.commit('$_nodes/ITEM_VARIANT', {mac: mac, variant: 'danger'})
          })
        })
      }
    },
    applyBulkDeregister () {
      const _this = this
      const macs = this.checkedRows.map(item => item.mac)
      if (macs.length > 0) {
        _this.$store.dispatch('$_nodes/deregisterBulkNodes', {items: macs}).then(response => {
          response.items.forEach(function (item, index, items) {
            _this.$store.commit('$_nodes/ITEM_VARIANT', {mac: item.mac, status: item.status})
            _this.$store.commit('$_nodes/ITEM_MESSAGE', {mac: item.mac, message: item.message})
          })
        }).catch(() => {
          macs.forEach(function (mac, index) {
            _this.$store.commit('$_nodes/ITEM_VARIANT', {mac: mac, variant: 'danger'})
          })
        })
      }
    },
    applyBulkReevaluateAccess () {
      const _this = this
      const macs = this.checkedRows.map(item => item.mac)
      if (macs.length > 0) {
        _this.$store.dispatch('$_nodes/reevaluateAccessBulkNodes', {items: macs}).then(response => {
          response.items.forEach(function (item, index, items) {
            _this.$store.commit('$_nodes/ITEM_VARIANT', {mac: item.mac, status: item.status})
            _this.$store.commit('$_nodes/ITEM_MESSAGE', {mac: item.mac, message: item.message})
          })
        }).catch(() => {
          macs.forEach(function (mac, index) {
            _this.$store.commit('$_nodes/ITEM_VARIANT', {mac: mac, variant: 'danger'})
          })
        })
      }
    },
    applyBulkRestartSwitchport () {
      const _this = this
      const macs = this.checkedRows.map(item => item.mac)
      if (macs.length > 0) {
        _this.$store.dispatch('$_nodes/restartSwitchportBulkNodes', {items: macs}).then(response => {
          response.items.forEach(function (item, index, items) {
            _this.$store.commit('$_nodes/ITEM_VARIANT', {mac: item.mac, status: item.status})
            _this.$store.commit('$_nodes/ITEM_MESSAGE', {mac: item.mac, message: item.message})
          })
        }).catch(() => {
          macs.forEach(function (mac, index) {
            _this.$store.commit('$_nodes/ITEM_VARIANT', {mac: mac, variant: 'danger'})
          })
        })
      }
    },
    applyBulkRole (role) {
      const _this = this
      const macs = this.checkedRows.map(item => item.mac)
      if (macs.length > 0) {
        macs.forEach(function (mac, index) {
          _this.$store.dispatch('$_nodes/roleNode', {mac: mac, category_id: role.category_id}).then(response => {
            _this.$store.commit('$_nodes/ITEM_VARIANT', {mac: mac, status: response.status})
            _this.$store.commit('$_nodes/ITEM_MESSAGE', {mac: mac, message: response.message})
          }).catch(() => {
            _this.$store.commit('$_nodes/ITEM_VARIANT', {mac: mac, variant: 'danger'})
          })
        })
      }
    },
    applyBulkBypassRole (role) {
      const _this = this
      const macs = this.checkedRows.map(item => item.mac)
      if (macs.length > 0) {
        macs.forEach(function (mac, index) {
          _this.$store.dispatch('$_nodes/bypassRoleNode', {mac: mac, bypass_role_id: role.category_id}).then(response => {
            _this.$store.commit('$_nodes/ITEM_VARIANT', {mac: mac, status: response.status})
            _this.$store.commit('$_nodes/ITEM_MESSAGE', {mac: mac, message: response.message})
          }).catch(() => {
            _this.$store.commit('$_nodes/ITEM_VARIANT', {mac: mac, variant: 'danger'})
          })
        })
      }
    },
    applyBulkViolation (violation) {
      const _this = this
      const macs = this.checkedRows.map(item => item.mac)
      if (macs.length > 0) {
        _this.$store.dispatch('$_nodes/applyViolationBulkNodes', {items: macs, vid: violation.id}).then(response => {
          response.items.forEach(function (item, index, items) {
            _this.$store.commit('$_nodes/ITEM_VARIANT', {mac: item.mac, status: item.status})
            _this.$store.commit('$_nodes/ITEM_MESSAGE', {mac: item.mac, message: item.message})
          })
        }).catch(() => {
          macs.forEach(function (mac, index) {
            _this.$store.commit('$_nodes/ITEM_VARIANT', {mac: mac, variant: 'danger'})
          })
        })
      }
    },
    onKeydown (event) {
      switch (true) {
        case (event.altKey && event.keyCode === 65): // ALT+A
          event.preventDefault()
          this.checkedRows = this.tableValues
          break
        case (event.altKey && event.keyCode === 78): // ALT+N
          event.preventDefault()
          this.checkedRows = []
          break
      }
    }
  },
  watch: {
    checkedRows (a, b) {
      this.checkedAll = (this.tableValues.length === a.length && a.length > 0)
      let _this = this
      let checkedRows = this.checkedRows
      this.items.forEach(function (item, index, items) {
        if (checkedRows.includes(item)) {
          _this.$store.commit('$_nodes/ITEM_VARIANT', {mac: item.mac, variant: 'info'})
        } else {
          _this.$store.commit('$_nodes/ITEM_VARIANT', {mac: item.mac, variant: ''})
          _this.$store.commit('$_nodes/ITEM_MESSAGE', {mac: item.mac, message: ''})
        }
      })
    },
    condition (a, b) {
      if (a !== b) this.clearChecked()
      try {
        if (a.values.length > 1 || a.values[0].values.length > 1) this.advancedMode = true
      } catch (e) {
        // noop
      }
    },
    requestPage (a, b) {
      if (a !== b) this.clearChecked()
    },
    currentPage (a, b) {
      if (a !== b) this.clearChecked()
    },
    pageSizeLimit (a, b) {
      if (a !== b) this.clearChecked()
    },
    visibleColumns (a, b) {
      if (a !== b) this.clearChecked()
    },
    query (a, b) {
      if (a !== b) {
        if (a) {
          let condition = JSON.parse(a)
          this.onSearch(condition)
        } else {
          // reset
          this.onSearch(undefined)
          this.condition = { op: 'and', values: [{ op: 'or', values: [{ field: this.fields[0].value, op: null, value: null }] }] }
        }
      }
    }
  },
  created () {
    this.$store.dispatch('config/getRoles')
    this.$store.dispatch('config/getViolations')
    this.pageSizeLimit = this.$store.state.$_nodes.searchPageSize
    while (true) {
      try {
        if (this.query) {
          this.condition = JSON.parse(this.query)
          break
        } else if (this.$store.state.$_nodes.searchQuery) {
          // Restore search parameters
          this.condition = this.$store.state.$_nodes.searchQuery
          break
        }
      } catch (e) {
        // noop
      }
      this.condition = { op: 'and', values: [{ op: 'or', values: [{ field: this.fields[0].value, op: null, value: null }] }] }
      break
    }
    // Restore visibleColumns, overwrite defaults
    if (this.$store.state.$_nodes.visibleColumns) {
      let visibleColumns = this.$store.state.$_nodes.visibleColumns
      this.columns.forEach(function (column, index, columns) {
        columns[index].visible = visibleColumns.includes(column.key)
      })
    }
    this.$store.dispatch('$_nodes/setSearchFields', this.searchFields)
    this.$store.dispatch('$_nodes/search', this.requestPage)
  },
  mounted () {
    document.addEventListener('keydown', this.onKeydown)
  },
  beforeDestroy () {
    document.removeEventListener('keydown', this.onKeydown)
  }
}
</script>
   
