<template>
    <li class="dropdown">
        <a href="javascript:void(0);" :class="[getNodeClass(node), getActiveNode(node)]" @click="nodeSelected(node,parent)" :title="node.itemname"><span class="span-menu">{{ node.text }}</span></a>
        <ul v-if="node.items && node.items.length" class="panel-collapse menu-folder" role="menu">
          <TreeView v-for="(child, index) in node.items" :key="index" :node="child" :parent="node" @node-selected="childSelected" :activeNode="activeNode" @update-active-node="updateActiveNode" />
        </ul>
    </li>
</template>
<style>
.span-menu { padding-left: 4px; }
a.selected-linker span { text-decoration: underline; }
</style>
<script>
export default {
  name: 'TreeView',
  props: {
    node: Object,
    parent: Object,
    activeNode: Object,
  },
  emits: ["node-selected"],
  methods: {
    getNodeClass(node) {
        let itemname = node?.itemname;
        if(!itemname || itemname.trim().length==0) return "fa fa-tasks";
        return "fa fa-desktop";
    },
    getActiveNode(node) {
      return this.activeNode == node ? "selected-linker" : "";
    },
    nodeSelected(curnode,parentnode) {
        this.$emit('node-selected',curnode,parentnode);
        this.$emit("update-active-node",curnode);
    },
    childSelected(curnode,parentnode) {
        this.$emit("node-selected",curnode,parentnode);
        this.$emit("update-active-node",curnode);
    },
  },
};
</script>
