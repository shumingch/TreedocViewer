<template>
  <b-button-group class="ml-1">
    <span v-b-tooltip.hover title="Collapse all">
      <b-btn :size="'sm'" @click='collapseAll()' :disabled='!canCollapse()'>&laquo;</b-btn>
    </span>
    <span v-b-tooltip.hover title="Collapse one level">
      <b-btn :size="'sm'" @click='collapse()' :disabled='!canCollapse()'>&lsaquo;</b-btn>
    </span>
    <span v-b-tooltip.hover title="Current expand level">
      <b-btn :size="'sm'" :disabled="true" class="expand_level">{{state.expandLevel}}</b-btn>
    </span>
    <span v-b-tooltip.hover title="Expand one level">
      <b-btn :size="'sm'" @click='expend()' :disabled='!canExpand()'>&rsaquo;</b-btn>
    </span>
    <span v-b-tooltip.hover title="Expand all">
      <b-btn :size="'sm'" @click='expendAll()' :disabled='!canExpand()'>&raquo;</b-btn>
    </span>
  </b-button-group>
</template>

<script lang="ts">
import { Component, Prop, Vue, Watch } from 'vue-property-decorator';
import { TreeNode } from '../models/Tree';

export class ExpandState {
  fullyExpand = false;
  moreLevel = false;

  constructor(
    public expandLevel = 1,
    public minLevel = 1,
  ) {}
}

@Component
export default class ExpandControl extends Vue {
  @Prop() state!: ExpandState;

  collapse() {
    if (this.state.expandLevel > this.state.minLevel)
      this.state.expandLevel--;
    this.state.fullyExpand = false;
  }

  collapseAll() {
    this.state.expandLevel = this.state.minLevel;
    this.state.fullyExpand = false;
  }

  canCollapse() {
    return this.state.expandLevel > this.state.minLevel || this.state.fullyExpand;
  }

  expend() {
    this.state.moreLevel = false;
    this.state.expandLevel++;
  }

  expendAll() {
    this.state.moreLevel = false;
    this.state.fullyExpand = true;
  }

  canExpand() {
    return this.state.moreLevel;
  }
}
</script>
<style scoped>
.expand_level {
  width: 16px;
  padding-left: 0px;
  padding-right: 0px;
  text-align: center;
}
</style>
