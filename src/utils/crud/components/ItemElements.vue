<template>
  <v-dialog persistent v-model="itemElements.show" max-width="800">
    <v-card>
      <v-card-title class="headline">{{ itemElements.title }}</v-card-title>
      <v-card-title>

        <!-- add/remove connection -->
        <template>
          <v-tooltip top>
            <v-btn class="white--text" fab small color="red" @click="removeMany()" slot="activator">
              <v-icon>delete</v-icon>
            </v-btn>
            <span>{{ $t('global.itemElements.buttons.removeMany') }}</span>
          </v-tooltip>
          <v-tooltip top>
            <v-btn class="white--text" fab small color="green" @click="addMany()" slot="activator">
              <v-icon>add</v-icon>
            </v-btn>
            <span>{{ $t('global.itemElements.buttons.addMany') }}</span>
          </v-tooltip>
        </template>

        <!-- Select statuses (added/no added) -->
        <template>
          <v-spacer></v-spacer>
          <v-autocomplete :label="$t('global.itemElements.status.title')" v-bind:items="statuses" v-model="selectedStatuses" single-line item-text="text" item-value="value"
            multiple chips></v-autocomplete>
        </template>

        <!-- Search in table -->
        <v-spacer></v-spacer>
        <v-text-field append-icon="search" :label="$t('global.itemElements.search')" single-line hide-details v-model="search"></v-text-field>
      </v-card-title>
      <!-- Table -->
      <v-data-table
        :disable-initial-sort="true"
        :must-sort="true"
        v-model="selected"
        :pagination.sync="pagination"
        select-all="black"
        :rows-per-page-items="[10, 25, { text: $t('global.itemElements.all'), value: -1 }]"
        light
        :headers="headers"
        :items="filteredItems"
        :no-results-text="$t('global.itemElements.noMatchingResults')"
        :no-data-text="$t('global.itemElements.noDataAvailable')"
        :rows-per-page-text="$t('global.itemElements.rowsPerPageText')"
      >
        <template slot="items" slot-scope="props">
          <td>
            <v-checkbox hide-details v-model="props.selected" color="black"></v-checkbox>
          </td>
          <!-- action buttons -->
          <td class="text-xs-center">
            <!-- add/remove connection (if soft deletes are enabled) -->
            <template>
              <v-tooltip top v-if="props.item.added == '<span hidden>1</span>Tak'">
                <v-btn fab small class="xs white--text" color="red" @click="remove(props.item.connectionId)" slot="activator">
                  <v-icon>delete</v-icon>
                </v-btn>
                <span>{{ $t('global.itemElements.buttons.remove') }}</span>
              </v-tooltip>
              <v-tooltip top v-else>
                <v-btn fab small class="xs white--text" color="green" @click="add(props.item.id)" slot="activator">
                  <v-icon>add</v-icon>
                </v-btn>
                <span>{{ $t('global.itemElements.buttons.add') }}</span>
              </v-tooltip>
            </template>
          </td>
          <!-- table fields -->
          <template v-for="(field, key) in props.item">
            <td v-if="!['id', 'connectionId', 'filterStatus'].includes(key)" class="text-xs-center" v-html="field" :key="key"></td>
          </template>
        </template>
        <template slot="pageText" slot-scope="{ pageStart, pageStop, itemsLength }">
          {{ $t('global.itemElements.records') }} {{ pageStart }} - {{ pageStop }} {{ $t('global.itemElements.from') }} {{ itemsLength }}
        </template>
      </v-data-table>
      <v-card-actions>
        <v-spacer></v-spacer>
        <v-btn color="black" flat="flat" @click.native="close()">{{ $t('global.itemElements.buttons.close') }}</v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
</template>
<script>
import {
  mapState, mapActions
} from 'vuex'

export default {
  props: ['details'],
  data () {
    return {
      selected: [],
      tmp: '',
      search: '',
      pagination: {
        sortBy: 'added',
        descending: true
      },
      selectedStatuses: [1, 0]
    }
  },
  computed: {
    ...mapState('crud', ['itemElements']),
    statuses () {
      return [
        {
          text: this.$t('global.itemElements.status.added'),
          value: 1
        },
        {
          text: this.$t('global.itemElements.status.noAdded'),
          value: 0
        }
      ]
    },
    headers () {
      const { columns } = this.itemElements
      const headers = columns.map((column) => {
        const header = {}
        header.text = column.header
        header.align = 'center'
        header.value = column.name
        return header
      })
      const actionHeader = [
        {
          text: this.$t('global.itemElements.fields.action'),
          align: 'center',
          sortable: false
        }
      ]
      const addedHeader = [
        {
          text: this.$t('global.itemElements.fields.added'),
          value: 'added'
        }
      ]
      return [...actionHeader, ...headers, ...addedHeader]
    },
    items () {
      const statusObject = this.itemElements.itemObject
      const { columns } = this.itemElements
      const connectionKeyName = this.itemElements.connectionKeyName || 'id'
      const dictionaryKeyName = this.itemElements.dictionaryKeyName || 'id'
      const items = this.itemElements.data.map((item) => {
        const rItem = {}
        rItem.id = item[dictionaryKeyName]
        for (const column of columns) {
          const prop = column.obj.split('.').reduce((object, property) => object[property] || '', item)
          rItem[column.name] = prop
        }
        if (typeof item[statusObject] !== 'undefined') {
          if (item[statusObject].length > 0) {
            rItem.added = '<span hidden>1</span>Tak'
            rItem.filterStatus = 1
            rItem.connectionId = item[statusObject][0][connectionKeyName]
          } else {
            rItem.added = '<span hidden>0</span>Nie'
            rItem.filterStatus = 0
          }
        }
        return rItem
      })
      return items
    },
    filteredItems () {
      const items = this.items.filter(item => this.selectedStatuses.includes(item.filterStatus))
      const phrases = this.search === '' ? [] : this.search.toLowerCase().split(' ')
      const filteredItems = items.filter((item) => {
        let found = true
        for (let i = 0; i < phrases.length; i++) {
          found = false
          for (const key in item) {
            let field = item[key]
            if (
              typeof field === 'string' ||
              field instanceof String ||
              typeof field === 'number'
            ) {
              field = field.toString().toLowerCase()
              if (field.includes(phrases[i])) {
                found = true
              }
            }
          }
          if (!found) break
        }
        return found
      })
      return filteredItems
    }
  },
  methods: {
    ...mapActions('crud', [
      'addItemElement',
      'removeItemElement',
      'addManyItemElements',
      'removeManyItemElements'
    ]),
    add (foreignId) {
      const obj = {}
      obj[this.itemElements.primaryId] = this.itemElements.id
      obj[this.itemElements.foreignId] = foreignId
      this.addItemElement([
        obj,
        this.$t('global.alerts.added'),
        this.$t('global.alerts.addError')
      ])
    },
    remove (id) {
      this.removeItemElement([
        id,
        this.$t('global.alerts.removed'),
        this.$t('global.alerts.removeError')
      ])
    },
    addMany () {
      const obj = this.selected
        .filter(item => item.connectionId === undefined)
        .map((item) => {
          const rItem = {}
          rItem[this.itemElements.primaryId] = this.itemElements.id
          rItem[this.itemElements.foreignId] = item.id
          return rItem
        })
      this.addManyItemElements([
        {
          items: obj
        },
        this.$t('global.alerts.added'),
        this.$t('global.alerts.addError')
      ])
      this.clearSelected()
    },
    removeMany () {
      const obj = this.selected
        .filter(item => item.connectionId !== undefined)
        .map(item => item.connectionId)
      this.removeManyItemElements([
        {
          ids: obj
        },
        this.$t('global.alerts.removed'),
        this.$t('global.alerts.removeError')
      ])
      this.clearSelected()
    },
    close () {
      this.itemElements.show = false
      this.$parent.itemElementsClosed()
    },
    clearSelected () {
      this.selected = []
    }
  }
}
</script>
<style scoped>
.item-elements-list {
  height: 400px;
  overflow-y: auto;
}
</style>
