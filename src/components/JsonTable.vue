<template>
  <div>
    <datatable v-bind="tableOpt">
      <div class="jtt-tbl-toolbar">
        <span v-b-tooltip.hover title="Expand children as columns">
          <b-btn size='sm' variant='outline-secondary' :pressed.sync='isExpanded'>
            <i class="fa fa-arrows-h"></i>
          </b-btn>
        </span>
        <b-button-group class="ml-1">
          <!-- We have to wrapper the button so that tooltip will work properly when it's disabled -->
          <!-- https://bootstrap-vue.js.org/docs/components/tooltip/ -->
          <span v-b-tooltip.hover title="Go back">
            <b-btn :size="'sm'" @click='tstate.back()' :disabled='!tstate.canBack()'>
              <i class="fa fa-arrow-left"></i>
            </b-btn>
          </span>
          <span v-b-tooltip.hover title="Go forward">
            <b-btn :size="'sm'" @click='tstate.forward()' :disabled='!tstate.canForward()'>
              <i class="fa fa-arrow-right"></i>
            </b-btn>
          </span>
        </b-button-group>
        <expand-control :state='expandState' />
        <json-path :tree-node="this.tstate ? this.tstate.selected : null" v-on:nodeClicked='nodeClicked'/>
        <!-- query: <b-form-input size='sm' :v-bind="tableOpt.query" /> -->
      </div>
    </datatable>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue, Watch } from 'vue-property-decorator';

import { DatatableOptions, Column, Query } from './Vue2DataTable';
import DataFilter from './DataFilter';
import thFilter from './th-Filter.vue';
import tdValue from './td-Value.vue';
import tdKey from './td-Key1.vue';
import JsonPath from './JsonPath.vue';
import TreeState from '../models/TreeState';
import JSONParser from '../parsers/JSONParser';
import ExpandControl, { ExpandState } from './ExpandControl.vue';
import { TDNode, TDNodeType } from 'treedoc';

const COL_VALUE = '@value';
const COL_NO = '#';
const COL_KEY = '@key';

@Component({
  components: {
    JsonPath,
    ExpandControl,
  },
})
export default class JsonTable extends Vue {
  tableOpt: DatatableOptions = {
    tblClass: 'table-bordered',
    pageSizeOptions: [5, 20, 50, 100, 200],
    columns: [],
    data: [],
    rawData: [],
    total: 0,
    query: { limit: 100, offset: 0 },
    xprops: { tstate: null },
  };
  defTableOpt!: any;
  // !! class based component, we have to initialized the data field, "undefined" won't be reactive. !!
  // https://github.com/vuejs/vue-class-component#undefined-will-not-be-reactive
  tstate: TreeState = new TreeState({});
  isExpanded = false;
  expandState = new ExpandState(0, 0);

  @Prop() private tableData!: TreeState | TDNode | object | string;
  @Prop() private options?: DatatableOptions;

  rebuildTable(val: TDNode) {
    if (!this.defTableOpt)  // backup for the first time, we have to intialize tableOpt attributes to make them reactive
      this.defTableOpt = this.tableOpt;

    this.defTableOpt.columns = [];
    this.tableOpt = { ...this.defTableOpt, ...(this.applyCustomOpts && this.options) };
    this.buildTable(val);
    this.queryData();
    this.tableOpt.xprops.tstate = this.tstate;
    this.tableOpt.xprops.expandState = this.expandState;
  }

  buildTable(val: TDNode) {
    this.tableOpt.rawData = [];

    if (!val)
      return;
    const ia = val.type === TDNodeType.ARRAY;
    const keyCol = ia ? COL_NO : COL_KEY;
    this.addColumn(keyCol, 0);

    if (val.children) {
      for (const v of val.children) {
        const row = {
          [keyCol]: ia ? Number(v.key) : v.key,
          [COL_VALUE]: v,
        };
        this.tableOpt.rawData.push(row);
        if (this.isExpanded && v && v.children) {
          for (const cv of v.children) {
            this.addColumn(cv.key);
            row[cv.key] = cv;
          }
        } else {
          this.addColumn(COL_VALUE, 1);
        }
      }
    }
  }

  addColumn(field: string, idx = this.tableOpt.columns.length) {
    const isKeyCol = idx === 0;
    const cols = this.tableOpt.columns;
    let col = cols.find(c => c.field === field);
    if (!col) {
      col = {
        field,
        visible: isKeyCol || !(this.applyCustomOpts && this.options!.columns),
      };
      cols.splice(idx, 0, col);
    }
    if (col.processed)
      return;

    col.title = col.title || field;
    col.sortable = true;
    col.thComp = col.thComp || thFilter;
    col.tdComp = col.tdComp || (isKeyCol ? tdKey : tdValue);
    col.processed = true;

    if (isKeyCol) {
      col.thClass = 'jsontable-min';
      col.tdClass = 'jsontable-min';
    }
  }

  defaultExpand(val: TDNode) {
    if (!val)
      return false;
    const cols = new Set<string>();
    let cellCnt = 0;
    if (val.children) {
      for (const v of val.children) {
        if (v && v.children) {
          for (const ck of Object.keys(v.children)) {
            cols.add(ck);
            cellCnt++;
          }
        }
      }
    }
    const totalCell = cols.size * val.getChildrenSize();
    // Limited number of cols due to performance reason
    return cols.size < 100 && cellCnt * 2 > totalCell;  // Fill ratio > 1/3
  }

  nodeClicked(data: TDNode) { this.tstate.select(data); }

  queryData() {
    const opt = this.tableOpt;
    opt.data = DataFilter.filter(opt.columns, opt.rawData, opt.query);
    opt.total = opt.rawData.length;
  }

  @Watch('query', {deep: true})
  watchQuery() { this.queryData(); }

  @Watch('isExpanded')
  watchIsExpanded() { this.rebuildTable(this.selected!); }

  @Watch('tableData', {immediate: true})
  watchTableData() {
    this.tstate = this.tableData && this.tableData instanceof TreeState ? this.tableData : new TreeState(this.tableData, new JSONParser());
  }

  @Watch('tstate.selected', {immediate: true})
  watchSelected(val: TDNode) {
    this.isExpanded = this.defaultExpand(val);
    this.tableOpt.query.offset = 0;
    this.expandState = new ExpandState(0, 0);
    this.rebuildTable(val);
  }

  @Watch('options', {immediate: true})
  optionsUpdated() { this.rebuildTable(this.selected!); }

  get selected(): TDNode | null {
    return this.tstate ? this.tstate.selected : null;
  }

  get applyCustomOpts() {
    return this.tstate.isInitialNodeSelected() && this.isExpanded && this.options;
  }

  get query() { return this.tableOpt.query; }
}
</script>

<style>
.jsontable-min {
  width:1%;
  /* white-space: nowrap; */
}
.table td, .table th {
  padding: .25rem;
}
pre {
  /* white-space: pre-wrap; */
  white-space: pre;
  word-wrap: break-word;
  margin-bottom: 0px;
}
.clearfix {
  margin-bottom: 0px !important;
}
.jtt-tbl-toolbar {
  display: flex;
  flex-wrap: wrap;
}
</style>
