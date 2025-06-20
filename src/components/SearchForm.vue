<template>
<div id="searchpanel" class="panel-body search-panel">
    <div class="row row-height">
        <div class="col-height col-md-3 col-label">
            <label for="groupname">{{ labels.groupname_label }}</label>
        </div>
        <div class="col-height col-md-3">
            <select ref="groupname" v-model="localData.groupname" class="form-control input-md" id="groupname">
              <option v-for="item in dataCategory.tgroup" :key="item.id" :value="item.id">{{item.id +" - "+ item.text}}</option>
            </select>
        </div>
        <div class="col-height col-md">
            <button @click="exploreClick" id="explorebutton" class="btn btn-dark btn-sm btn-explore"><i class="fa fa-search fa-btn-icon" aria-hidden="true"></i>{{ labels.explore_button }}</button>
        </div>
    </div>
</div>
</template>
<script>
import { ref } from 'vue';

const defaultData = {
  groupname: "",
};

export default {
  props: {
    labels: Object,
    formData: Object,
    dataCategory: Object
  },
  emits: ["data-explore"],
  setup(props) {
    const localData = ref({ ...defaultData, ...props.formData });
    return { localData };
  },
  methods: {
    reset(newData) {
      if(newData) this.localData = {...newData};
    },
    exploreClick() {
      this.search();
    },
    search() {
      this.$emit('data-explore',this.localData);
    },
  }
};
</script>
