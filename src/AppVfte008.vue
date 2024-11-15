<!-- App.vue -->
<template>
  <div id="fswaitlayer" class="fa fa-spinner fa-spin"></div>
  <div class="pt-page pt-page-current pt-page-controller search-pager">
    <PageHeader ref="pageHeader" :labels="labels" pid="vfte008" version="1.0.0" showLanguage="true" @language-changed="changeLanguage" :multiLanguages="multiLanguages" :build="buildVersion" />
    <SearchForm ref="searchForm" :labels="labels" :dataCategory="dataCategory" @data-explore="dataExplore" />
  </div>
  <EntryForm ref="entryForm" :labels="labels" :dataCategory="dataCategory" @data-saved="dataSaved" />
</template>
<script>
import { ref } from 'vue';
import $ from "jquery";
import { PageHeader } from '@willsofts/will-control';
import SearchForm from '@/components/SearchForm.vue';
import EntryForm from '@/components/EntryForm.vue';
import { getLabelModel, getMultiLanguagesModel } from "@willsofts/will-app";
import { DEFAULT_CONTENT_TYPE, getDefaultLanguage, setDefaultLanguage, getApiUrl } from "@willsofts/will-app";
import { startApplication, serializeParameters } from "@willsofts/will-app";

const buildVersion = process.env.VUE_APP_BUILD_DATETIME;
export default {
  components: {
    PageHeader, SearchForm, EntryForm
  },
  setup() {
    const dataChunk = {};
    const dataCategory = {
      tgroup: [],
      tprog: [],
      tkpermit: []
    };
    let labels = ref(getLabelModel());
    let alreadyLoading = ref(false);
    const multiLanguages = ref(getMultiLanguagesModel());
    return { buildVersion, multiLanguages, labels, dataCategory, dataChunk, alreadyLoading };
  },
  mounted() {
    console.log("App: mounted ...");
    this.$nextTick(async () => {
      //ensure ui completed then invoke startApplication 
      startApplication("vfte008",(data) => {
        this.multiLanguages = getMultiLanguagesModel();
        this.messagingHandler(data);
        this.loadDataCategories(!this.alreadyLoading,() => {
          this.$refs.pageHeader.changeLanguage(getDefaultLanguage());
        });
      });
    });
  },
  methods: {
    messagingHandler(data) {
      console.log("messagingHandler: data",data); 
    },
    changeLanguage(lang) {
      setDefaultLanguage(lang);
      let labelModel = getLabelModel(lang);
      this.labels = labelModel;
      this.resetDataCategories(lang);
    },
    loadDataCategories(loading,callback) {
      console.log("loadDataCategories: loading",loading);
      if(!loading) return;
      let jsondata = {names: ["tgroup", "tprog", "tkpermit"]};
      let formdata = serializeParameters(jsondata);
      $.ajax({
        url: getApiUrl()+"/api/category/lists",
        data: formdata.jsondata,
        headers : formdata.headers,
        type: "POST",
        dataType: "json",
        contentType: DEFAULT_CONTENT_TYPE,
        error : function(transport,status,errorThrown) {
          console.error("loadDataCategories: error: status",status,"errorThrown",errorThrown);
        },
        success: (data) => {
          this.alreadyLoading = true;
          console.log("loadDataCategories: success",data);
          if(data.body) {
            for(let item of data.body) {
              if(item.category && item.resultset && item.resultset.rows) {
                this.dataChunk[item.category] = item.resultset.rows;
              }
            }
            console.log("data chunk",this.dataChunk);
            this.resetDataCategories();
            if(callback) callback();
          }
        }
      });	
    },
    resetDataCategories(lang) {
      if(!lang) lang = getDefaultLanguage();
      if(!lang || lang.trim().length==0) lang = "EN";
      let tgroup;
      let tprog;
      let tkpermit;
      let tk_category = this.dataChunk["tgroup"];
      if(tk_category) {
        tgroup = tk_category.map((item) => { return { id: item.groupname, text: "TH"==lang?item.nameth:item.nameen } });
      }
      tk_category = this.dataChunk["tprog"];
      if(tk_category) {
        tprog = tk_category.map((item) => { return { id: item.programid, text: "TH"==lang?item.prognameth:item.progname } });
      }
      tk_category = this.dataChunk["tkpermit"];
      if(tk_category) {
        tkpermit = tk_category.map((item) => { return { id: item.typeid, text: "TH"==lang?item.nameth:item.nameen } });
      }
      if(tgroup) this.dataCategory.tgroup = tgroup;
      if(tprog) this.dataCategory.tprog = tprog;
      if(tkpermit) this.dataCategory.tkpermit = tkpermit;
    },
    dataExplore(data) {
      //listen action from search form
      console.log("App: dataExplore ",data);
      this.$refs.entryForm.retrieveRecord(data);
    },
    dataSaved(data,response) {
      //listen action from entry form when after saved
      console.log("App: record saved");
      console.log("data",data,"response",response);
    },
  }
};
</script>