<script>
import { sizeFormatter } from '../utils/mixins';
import GirderBreadcrumb from './Breadcrumb.vue';

const GIRDER_FOLDER_ENDPOINT = 'folder';
const GIRDER_ITEM_ENDPOINT = 'item';

export default {
  components: {
    GirderBreadcrumb,
  },
  mixins: [sizeFormatter],
  props: {
    location: {
      type: Object,
      required: true,
      validator: val => val._modelType && val._id,
    },
    selectEnabled: {
      type: Boolean,
      default: true,
    },
    multiSelectEnabled: {
      type: Boolean,
      default: false,
    },
    uploadEnabled: {
      type: Boolean,
      default: false,
    },
    newItemEnabled: {
      type: Boolean,
      default: true,
    },
    newFolderEnabled: {
      type: Boolean,
      default: true,
    },
  },
  inject: ['girderRest'],
  data() {
    return {
      pagination: {
        rowsPerPage: 10,
        page: 1,
      },
      refreshCounter_: 0,
      rows: [],
      rowsLoading: false,
      selected: [],
    };
  },
  computed: {
    loading() {
      return (this.$refs.breadcrumb && this.$refs.breadcrumb.loading) || this.rowsLoading;
    },
    totalItems() {
      return this.counts.nFolders + this.counts.nItems;
    },
  },
  asyncComputed: {
    counts: {
      default: { nFolders: 0, nItems: 0 },
      async get() {
        const { _modelType, _id } = this.location;
        const endpoint = `${_modelType}/${_id}/details`;
        const { data } = await this.girderRest.get(endpoint);
        return {
          nFolders: data.nFolders || 0,
          nItems: data.nItems || 0,
        };
      },
      watch() {
        return [
          this.refreshCounter_,
          this.location,
        ];
      },
    },
  },
  watch: {
    location(newval, oldval) {
      // force reset pagination when location changes.
      if (newval._id !== oldval._id) {
        this.pagination.page = 1;
      }
    },
    selected(newval) {
      this.$emit('selection-changed', newval);
    },
    async counts() {
      this.rows = await this.fetchPaginatedRows();
    },
    // Not triggered by location pagination reset because deep==false
    async pagination() {
      this.rows = await this.fetchPaginatedRows();
    },
  },
  methods: {
    toggleAll() {
      if (this.selected.length === this.rows.length) {
        this.selected = [];
      } else {
        this.selected = this.rows.slice();
      }
    },
    changeLocation(item) {
      const { _modelType, _id, name } = item;
      if ((this.location._id !== _id || this.location._modelType !== _modelType) &&
          _modelType !== 'item') {
        this.$emit('update:location', { _modelType, _id, name });
      }
    },
    refresh() {
      this.refreshCounter_ += 1;
    },
    async fetchPaginatedRows() {
      if (!this.counts.nFolders && !this.counts.nItems) {
        return [];
      }
      this.rowsLoading = true;
      const { counts, location, pagination } = this;
      const { page, rowsPerPage } = pagination;
      const folderParams = {
        parentType: location._modelType,
        parentId: location._id,
        limit: rowsPerPage >= 0 ? rowsPerPage : null,
        offset: (page - 1) * rowsPerPage,
      };
      const itemLimit = counts.nFolders > folderParams.offset
        // if there are any folders on the current page,
        // the numer of items to fetch is based on the number of folders
        ? rowsPerPage - (counts.nFolders - folderParams.offset)
        // else the page will be comprised of only items
        : rowsPerPage;
      const itemOffset = folderParams.offset - counts.nFolders;
      const itemParams = {
        folderId: location._id,
        limit: rowsPerPage >= 0 ? itemLimit : null,
        offset: itemOffset > 0 ? itemOffset : 0,
      };
      const promises = [];
      promises.push(this.girderRest.get(GIRDER_FOLDER_ENDPOINT, { params: folderParams }));
      // a limit of < 0 signifies the current page only includes folders
      // a limit of null signifies no pagination: fetch all entities
      if ((itemParams.limit > 0 || itemParams.limit === null) && location._modelType === 'folder') {
        promises.push(this.girderRest.get(GIRDER_ITEM_ENDPOINT, { params: itemParams }));
      }
      const responses = (await Promise.all(promises)).map(response => response.data);
      const rows = [].concat.apply(...responses).map(item => ({
        ...item,
        size: item.size ? this.formatSize(item.size) : '',
        icon: item._modelType in this.$vuetify.icons
          ? item._modelType
          : 'file',
      }));
      this.rowsLoading = false;
      return rows;
    },
  },
};
</script>

<template lang="pug">
v-data-table.girder-file-browser-component.elevation-1(
    select-all,
    :headers-length="4",
    v-model="selected",
    :pagination.sync="pagination",
    :items="rows",
    :total-items="totalItems",
    :loading="loading ? 'accent' : false",
    item-key="_id")

  //- Header Slot
  template(slot="headers", slot-scope="props")
    tr.secondary.lighten-5
      th.pl-3.pr-0(width="1%", v-if="selectEnabled")
        v-checkbox.secondary--text.text--darken-1.pr-2(
            color="accent",
            hide-details,
            :input-value="props.all",
            :indeterminate="selected.length > 0 && !props.all",
            @click.native="toggleAll")
      th.pl-3
        girder-breadcrumb(
            ref="breadcrumb",
            :location="location",
            @crumbclick="changeLocation")
      th.pr-0.align-right(width="1%")
        v-btn(flat,
            small,
            color="secondary darken-2",
            v-if="newFolderEnabled",
            @click="$emit('click:newfolder')")
          v-icon.mdi-24px.mr-1(left, color="accent") {{  $vuetify.icons.folderNew }}
          span.hidden-xs-only New Folder
      th.pl-0.pr-1.align-right(width="1%")
        v-btn(flat,
            small,
            color="secondary darken-2",
            v-if="newItemEnabled",
            @click="$emit('click:newitem')")
          v-icon.mdi-24px.mr-1(left, color="accent") {{  $vuetify.icons.fileNew }}
          span.hidden-xs-only New Item

  //- Table Row Slot
  template(slot="items", slot-scope="props")
    tr.itemRow(:active="props.selected",
        @click="if (selectEnabled) props.selected = !props.selected;",
        :key="props.index")
      td.pl-3.pr-0(v-if="selectEnabled")
        v-checkbox.secondary--text.text--darken-1.pr-2(
            :input-value="props.selected", accent, hide-details)
      td.pl-3(colspan="2")
        span.text-container.secondary--text.text--darken-3.nobreak(
            :class="{selectable: props.item._modelType !== 'item'}",
            @click.stop="changeLocation(props.item)")
          v-icon.pr-2(:color="props.selected ? 'accent' : ''") {{ $vuetify.icons[props.item.icon] }}
          | {{ props.item.name }}
      td.text-xs-right.secondary--text.text--darken-3.nobreak {{ props.item.size }}
</template>

<style lang="scss" scoped>
.girder-file-browser-component.elevation-1 {
  box-shadow: none !important;

  .selectable {
    opacity: 0.8;

    &:hover {
      opacity: 1;
      cursor: pointer;
    }
  }

  .v-table {
    th {
      button.v-btn {
        min-width: 0;
        margin: 0;
      }
    }

    tr {
      &.itemRow[active],
      &.itemRow:hover {
        // $light-blue.lighten-5
        background: #e1f5fe !important;
      }

      &.secondary {
        border-color: inherit !important;
      }

      .v-input--checkbox {
        border-right: 1.5px solid;
      }

      .text-container i {
        vertical-align: bottom;
      }

      .nobreak {
        white-space: nowrap;
      }
    }
  }
}
</style>

<style lang="scss">
.girder-file-browser-component {
  .theme--light.v-icon {
    color: inherit;
  }

  .v-datatable__progress .v-progress-linear {
    position: absolute;
  }
}
</style>