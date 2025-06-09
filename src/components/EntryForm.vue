<template>
<div id="entrypanel" class="panel-body">
  <hr class="entry-horizontal"/>
  <div class="row data-layer">
    <div id="menubarcolumnlayer" class="col-md-7">
      <div id="menubarlayer" class="menubar-layer">
        <ul id="menuitemlist" class="nav sidebar-nav menu-folder" role="menu" v-show="hasMenu">
          <TreeView ref="treeView" :node="menuData" :parent="menuParent" @node-selected="treeSelected" :activeNode="activeNode" @update-active-node="updateActiveNode" />
        </ul>
      </div>
    </div>
    <div id="itemlayer" class="col-md">
        <div class="row">
            <button id="tasklinker" @click="newTaskClick" class="btn btn-dark btn-sm ctrl-linker fa fa-tasks" title="New Task" :disabled="canUpdate"></button>
            &nbsp;
            <button id="inserttasklinker" @click="insertTaskClick" class="btn btn-dark btn-sm ctrl-linker fa fa-indent" title="Insert Task" :disabled="canUpdate"></button>
            &nbsp;
            <button id="itemlinker" @click="newItemClick" class="btn btn-dark btn-sm ctrl-linker fa fa-desktop" title="New Item" :disabled="canUpdate"></button>
            &nbsp;
            <button id="deletelinker" @click="deleteClick" class="btn btn-dark btn-sm ctrl-linker fa fa-trash" title="Delete" :disabled="canDelete"></button>
        </div>
        <div id="entryformpanel">
            <div class="row">
                <fieldset id="propertyfieldset">
                    <legend><label class="lclass" id="properties_label">{{ labels.properties_label }}</label></legend>
                </fieldset>
            </div>
            <div class="row row-heighter center-block">
                <div class="col-md-3 col-height col-label">
                    <label for="progid" class="control-label">{{ labels.progid_label }}</label>
                </div>
                <div class="col-md-5 col-height">
                  <div class="input-group has-validation" :class="{'has-error': v$.progid.$error}">
                    <input type="text" ref="progid" v-model="itemData.progid" id="progid" class="form-control input-md alert-input input-entry" maxlength="20" :disabled="isDisabledItemField" /> 
                  </div>
                </div>
            </div>
            <div class="row row-heighter center-block">
                <div class="col-md-3 col-height col-label">
                    <label for="progtitle" class="control-label">{{ labels.progtitle_label }}</label>
                </div>
                <div class="col-md-9 col-height">
                  <div class="input-group has-validation" :class="{'has-error': v$.progtitle.$error}">
                    <input ref="progtitle" type="text" v-model="itemData.progtitle" id="progtitle" class="form-control input-md alert-input" maxlength="100" />
                  </div>
                </div>
            </div>
            <div class="row">
                <fieldset id="permissionfieldset">
                    <legend><label id="permissions_label" class="lclass">{{ labels.permissions_label }}</label></legend>
                </fieldset>
            </div>        
            <div id="permitinfolayer">
                <div v-for="(permit, index) in permitLists()" :key="index" class="row row-heighter">
                    <div v-for="item in permit" :key="item.id" class="col-height col-md-4">
                        <div class="checkbox form-check">
                            <input type="checkbox" :id="getBoxId(item)" :true-value="true" :false-value="false" v-model="itemData.permits[item.id]" class="form-control input-md form-check-input permit-checkbox" :disabled="isDisabledItemField" />
                            <label :for="getBoxId(item)" class="form-check-label">{{item.text}}</label>
                        </div>
                    </div>
                </div>
            </div>
            <div class="row row-heighter center-block">
                <div class="col-md-3 col-height col-label">
                    <label id="parameters_label" class="control-label">{{ labels.parameters_label }}</label>
                </div>
                <div class="col-md-9 col-height">
                    <input type="text" class="form-control input-md input-entry" id="parameters" v-model="itemData.parameters" maxlength="80" :disabled="isDisabledItemField" />
                </div>
            </div>
        </div>
        <div id="buttoncontrollayer" class="row">
            <div class="col-md-2">
              <button ref="updatebutton" id="updatebutton" class="btn btn-dark btn-sm" @click="updateClick" :disabled="canUpdate"><em class="fa fa-pencil fa-btn-icon"></em>{{ labels.update_button }}</button>
            </div>
            <div class="col-md pull-right text-right">
                <button ref="savebutton" id="savebutton" class="btn btn-dark btn-sm" @click="saveClick" :disabled="disabledKeyField"><em class="fa fa-save fa-btn-icon"></em>{{ labels.save_button }}</button>
                &nbsp;
                <button ref="cancelbutton" id="cancelbutton" class="btn btn-dark btn-sm" @click="cancelClick"><em class="fa fa-close fa-btn-icon"></em>{{ labels.cancel_button }}</button>
              </div>
        </div>
    </div>
  </div>
</div>
<teleport to="#taskdialog">
  <TaskForm ref="taskForm" :labels="labels" :dataCategory="dataCategory" />
</teleport>
<teleport to="#itemdialog">
  <ItemForm ref="itemForm" :labels="labels" :dataCategory="dataCategory" />
</teleport>
</template>
<script>
import { ref, computed, watch } from 'vue';
import { useVuelidate } from '@vuelidate/core';
import { required, helpers } from '@vuelidate/validators';
import $ from "jquery";
import { DEFAULT_CONTENT_TYPE, getApiUrl, disableControls }  from '@willsofts/will-app';
import { startWaiting, stopWaiting, submitFailure, detectErrorResponse, serializeParameters }  from '@willsofts/will-app';
import { confirmSave, confirmCancel, successbox, alertbox, alertmsg, confirmDialogBox } from '@willsofts/will-app';
import TaskForm from './TaskForm.vue';
import ItemForm from './ItemForm.vue';
import TreeView from './TreeView.vue';

const APP_URL = "/api/sfte008";
const defaultData = {
  groupname: "",
  menutext: "",
};

const defaultItemData = {
	progid: "",
	progtitle: "",
	parameters: "",
	permits: {},
};

export default {
  components: {
    TaskForm, ItemForm, TreeView
  },
  props: {
    modes: Object,
    labels: Object,
    dataCategory: Object
  },
  emits: ["data-saved"],
  setup(props) {
    const localData = ref({ ...defaultData }); 
    const disabledKeyField = ref(true);
    const itemData = ref({...defaultItemData});
    const menuData = ref({});
    const menuParent = ref(null);
    const activeNode = ref(null);
    const currentNode = ref(null);
    const parentNode = ref(null);
    const permits = ref({});
    const reqalert = ref(props.labels.empty_alert);
    const requiredField = () => { 
      let itemname = currentNode?.value?.itemname;
      if(!itemname) itemname = "";
      let disabled = disabledKeyField.value || itemname.trim().length == 0;      
      return disabled ? false : helpers.withMessage(reqalert, required); 
    };
    const requiredMessage = () => {
      return helpers.withMessage(reqalert, required);
    }
    const validateRules = computed(() => { 
      return {
        progid: { required: requiredField() },
        progtitle: { required: requiredMessage() },
      } 
    });
    //this must specified: $scope: false in order to prevent validate all nested components validation
    const v$ = useVuelidate(validateRules, itemData, { $lazy: true, $autoDirty: true, $scope: false });
    return { v$, localData, itemData, menuData, menuParent, activeNode, disabledKeyField, permits, currentNode, parentNode, reqalert };
  },
  created() {
    watch(this.$props, (newProps) => {      
      this.reqalert = newProps.labels.empty_alert;
      let sourceAry = [];
      let programs = newProps.dataCategory.tprog;
      programs.forEach((element) => {
        sourceAry.push({ label: element.id+" "+element.text, value: element.id, text: element.text });
      });
      $("#progid").autocomplete("option","source",sourceAry);
      let permits = {};
      newProps.dataCategory.tkpermit.forEach(item => { permits[item.id] = false });
      this.permits = permits;
    });
  },
  computed: {
    hasMenu() { return this.menuData.text && this.menuData.text.length > 0; },
    isDisabledItemField() { 
      let itemname = this.currentNode?.itemname;
      if(!itemname) itemname = "";
      return this.disabledKeyField || itemname.trim().length == 0; 
    },
    canUpdate() { return !this.currentNode; },
    canDelete() { return !this.currentNode || !this.parentNode; },
  },
  mounted() {
    this.$nextTick(() => {
      let sourceAry = [];
      let programs = this.$props.dataCategory.tprog;
      programs.forEach((element) => {
        sourceAry.push({ label: element.id+" "+element.text, value: element.id, text: element.text });
      });
      console.log("onmounted: nextTick sourceAry",sourceAry); 
      $("#progid").autocomplete({
        delay: 500, 
        source: sourceAry,
        select : (event, ui) => {
          console.log("on select: ",ui.item);
          this.itemData.progid = ui.item.value;
          this.itemData.progtitle = ui.item.text;
        },
      });
      $("#progid").autocomplete("widget").addClass("autocomplete-fixed-height");
    });
  },
  methods: {
    getBoxId(item) { return "permitbox"+item.id; },
    permitLists() {
      let results = [];
      let chunkSize = 3;
      let tkpermit = this.$props.dataCategory.tkpermit;
      for (let i = 0; i < tkpermit.length; i += chunkSize) {
          results.push(tkpermit.slice(i, i + chunkSize));
      }
      return results;
    },
    reset(newData) {
      if(newData) {
        let permits = {};
        this.$props.dataCategory.tkpermit.forEach(item => { permits[item.id] = false });
        permits = {...permits, ...newData?.permits};
        this.permits = permits;
        this.itemData = {...newData, permits: permits};
      }
    },
    async saveClick() {
      console.log("click: save");
      disableControls($("#savebutton"));
      this.startSaveRecord();
    },
    cancelClick() {
      console.log("click: cancel");
      disableControls($("#cancelbutton"));
      this.startCancelRecord();
    },
    async validateForm(focusing=true) {
      const valid = await this.v$.$validate();
      console.log("data:",this.itemData);
      console.log("validate form: valid",valid);
      console.log("error:",this.v$.$errors);
      if(!valid) {
        if(focusing) {
          this.focusFirstError();
        }
        return false;
      }
      return true;
    },
    focusFirstError() {
      if(this.v$.$errors && this.v$.$errors.length > 0) {
        let input = this.v$.$errors[0].$property;
        let el = this.$refs[input];
        if(el) el.focus(); //if using ref
        else $("#"+input).trigger("focus"); //if using id
      }
    },
    hasItemname() {
      if(this.currentNode) {
          let itemname = this.currentNode?.itemname;
          return itemname && itemname.trim().length > 0;
      }
      return false;
    },
    async updateClick() {
      let valid = await this.validateForm();
      if(!valid) return;
      if(this.hasItemname()) {
        this.currentNode.itemname = this.itemData.progid;
        this.currentNode.text = this.itemData.progtitle;
        this.currentNode.parameters = this.itemData.parameters;
        this.currentNode.permits = {...this.itemData.permits};
      } else {
        this.currentNode.text = this.itemData.progtitle;
      }
      this.updateSuccess();
      let $entry = $("#entryformpanel");
      $("input",$entry).removeClass("is-invalid");
      $(".alert-input",$entry).parent().removeClass("has-error");
    },
    deleteClick() {
      if(this.currentNode) {
        if(!this.parentNode) {
          alertmsg("QS0124","Can not remove root node");
        } else {
          console.log("removeNode:",this.parentNode);
          this.confirmRemoveNode([this.currentNode.text],() => {
            let items = this.parentNode.items.filter((element) => element != this.currentNode);
            this.parentNode.items = items;
            console.log("removeNode:",this.parentNode);
            this.itemData = {...defaultItemData};
            this.activeNode = null;
            this.currentNode = null;
          });
        }
      }
    },
    resetRecord() {
      this.reset(defaultData);      
      this.v$.$reset();
      this.disabledKeyField = true;
      this.menuData = {};
      this.itemData = {...defaultItemData};
      this.activeNode = null;
      this.currentNode = null;
      this.parentNode = null;
    },
    startCancelRecord() {
      confirmCancel(() => {
        this.resetRecord();
      });
    },
    startSaveRecord() {
      let menuDoc = this.menuData;
      console.log("menuDoc",menuDoc);
      this.localData.menutext = JSON.stringify(menuDoc);
      let childs = menuDoc?.items?.length;
      if(!childs || childs<=0) this.localData.menutext = "";
      console.log("localData",this.localData);
      confirmSave(() => {
        this.saveRecord(this.localData);
      });
    },
    retrieveRecord(dataKeys) {
      if(!dataKeys.groupname || dataKeys.groupname.trim().length==0) return;
      let jsondata = {ajax: true};
      let formdata = serializeParameters(jsondata,dataKeys);
      startWaiting();
      $.ajax({
        url: getApiUrl()+APP_URL+"/retrieve",
        data: formdata.jsondata,
        headers : formdata.headers,
        type: "POST",
        dataType: "json",
        contentType: DEFAULT_CONTENT_TYPE,
        error : function(transport,status,errorThrown) {
          console.error("retrieveRecord: error: status",status,"errorThrown",errorThrown);
          submitFailure(transport,status,errorThrown);
        },
        success: (data) => {
          stopWaiting();
          console.log("retrieveRecord: success",data);
          if(detectErrorResponse(data)) return;
          this.localData = { groupname: data?.body?.dataset?.groupname, menutext: data?.body?.dataset?.menutext };
          this.showMenuTree(data);
          this.disabledKeyField = false;
          this.itemData = {...defaultItemData};
          this.activeNode = null;
          this.currentNode = null;
          this.parentNode = null;
          this.v$.$reset();
        }
      });	
    },
    saveRecord(dataRecord) {
        let jsondata = {ajax: true};
        let formdata = serializeParameters(jsondata,dataRecord);
        startWaiting();
        $.ajax({
          url: getApiUrl()+APP_URL+"/update",
          data: formdata.jsondata,
          headers : formdata.headers,
          type: "POST",
          dataType: "json",
          contentType: DEFAULT_CONTENT_TYPE,
          error : function(transport,status,errorThrown) {
            console.error("error: status",status,"errorThrown",errorThrown);
            submitFailure(transport,status,errorThrown);
          },
          success: (data) => {
            stopWaiting();
            console.log("success",data);
            if(detectErrorResponse(data)) return;
            successbox(() => {
              //reset data for new record insert
              this.resetRecord();
              this.$emit('data-saved',dataRecord,data);
            });
          }
      });
    },
    newTaskClick() {
      this.newTaskInfo(false,this.$refs.taskForm);
    },
    insertTaskClick() {
      this.newTaskInfo(true,this.$refs.taskForm);
    },
    newItemClick() {
      this.newItemInfo(this.$refs.itemForm);
    },         
    showMenuTree(data) {
      let menu_texts = data?.body?.dataset?.menutext;
      if(!menu_texts || menu_texts.trim().length==0) {
        let text = $("option:selected",$("#groupname")).text();
        menu_texts = "{\"text\":\""+text+"\"}";
      }
      this.menuData = JSON.parse(menu_texts);
      console.log("menuData:",this.menuData);
      $("input[type=text]",$("#entryformpanel")).removeAttr("readonly");
    },
    treeSelected(curnode,parentnode) {
      console.log("tree selected: curnode",curnode,"parent",parentnode);
      this.currentNode = curnode;
      this.parentNode = parentnode;
      this.itemData = {progid: curnode?.itemname, progtitle: curnode?.text, parameters: curnode?.parameters, permits: {...this.permits, ...curnode?.permits} };
      this.v$.$reset();
    },
    updateActiveNode(node) { 
      this.activeNode = node; 
    },
    updateSuccess(callback,params) {
      alertbox("QS0104",callback,null,params);
    },
    confirmRemoveNode(params, okFn, cancelFn,  width, height) {
      if(!confirmDialogBox("QS0034",params,"Do you want to remove this node?",okFn,cancelFn,width,height)) return false;
      return true;
    },
    newTaskInfo(inserted,taskForm) {
      if(!this.currentNode) return;	
      taskForm.show((data) => { 
        let curnode = {text: data.progtitle};
        console.log("newItemInfo: data",data,"curnode",curnode);
        let rootTag = this.parentNode;
        if(inserted) {
          if(!rootTag) {
            let curitems = this.currentNode?.items;
            if(!curitems) {
              curitems = [];
              this.currentNode.items = curitems;
            }
            curitems.push(curnode);	
            //console.log("Task-1: cur items",curitems);
          } else {
            let parentitems = this.parentNode?.items;
            if(!parentitems) {
              parentitems = [];
              this.parentnode.items = parentitems;
            }
            if(parentitems.length==0) {
              parentitems.push(curnode);
            } else {
              let index = parentitems.findIndex(element => element == this.currentNode);
              if(index !== -1) {
                parentitems.splice(index+1,0,curnode);
              }
            }
            //console.log("Task-2: parent items",parentitems);
          }
        } else {
          let curitems = this.currentNode?.items;
          if(!curitems) {
            curitems = [];
            this.currentNode.items = curitems;
          }
          curitems.push(curnode);
          //console.log("Task-3: cur items",curitems);
        }		
        taskForm.hideDialog();
        console.log("Task-New: menuData",this.menuData);
      });
    },
    newItemInfo(itemForm) {
      if(!this.currentNode) return;
      itemForm.show((data) => {
        let curnode = {itemname: data.progid, text: data.progtitle, parameters: data.parameters, permits: data.permits};
        console.log("newItemInfo: data",data,"curnode",curnode);
        let itemname = this.currentNode.itemname;
        if(itemname && itemname!="") {
          let parentitems = this.parentNode?.items;
          if(!parentitems) {
            parentitems = [];
            this.parentNode.items = parentitems;
          }
          if(parentitems.length==0) {
            parentitems.push(curnode);
          } else {
            let index = parentitems.findIndex(element => element == this.currentNode);
            if(index !== -1) {
              parentitems.splice(index+1,0,curnode);
            }
          }
          //console.log("Item-1: parent items",parentitems);
        } else {
          let curitems = this.currentNode?.items;
          if(!curitems) {
            curitems = [];
            this.currentNode.items = curitems;
          }
          curitems.push(curnode);
          //console.log("Item-2: parent items",curitems);
        }
        itemForm.hideDialog();
        console.log("Task-New: menuData",this.menuData);
      });
    },
  }
};
</script>
