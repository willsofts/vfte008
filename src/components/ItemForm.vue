<template>
  <DialogForm ref="dialogForm" id="item_dialog_layer">
    <template v-slot:header>
      <h4 class="modal-title">{{ labels.item_dialog_title }}</h4>
    </template>
    <template v-slot:default>
        <div class="row row-height">
          <div class="col-md-3 col-height col-label">
            <label for="progiddialog">{{ labels.progid_label }}</label>
          </div>
          <div class="col-md-4 col-height">
            <div class="input-group has-validation" :class="{'has-error': v$.progid.$error}">
              <input type="text" ref="progiddialog" v-model="localData.progid" id="progiddialog" class="form-control input-md input-item" maxlength="20" /> 
            </div>
          </div>
        </div>
        <div class="row row-height">
          <div class="col-md-3 col-height col-label">
            <label for="progtitledialog">{{ labels.progtitle_label }}</label>
          </div>
          <div class="col-md-9 col-height">
            <div class="input-group has-validation" :class="{'has-error': v$.progtitle.$error}">
              <input ref="progtitledialog" type="text" v-model="localData.progtitle" id="progtitledialog" class="form-control input-md input-item" maxlength="100" />
            </div>
          </div>
        </div>
        <div class="row">
          <fieldset id="permitfieldset">
            <legend><label class="lclass" id="permissionsdialog_label">{{ labels.permissions_label }}</label></legend>
          </fieldset>
        </div>					
        <div id="permitinfolayerdialog">
            <div v-for="(permit, index) in permitLists()" :key="index" class="row row-heighter">
                <div v-for="item in permit" :key="item.id" class="col-height col-md-4">
                    <div class="checkbox form-check">
                        <input type="checkbox" :id="getBoxId(item)" :true-value="true" :false-value="false" v-model="localData.permits[item.id]" class="form-control input-md form-check-input"/>
                        <label :for="getBoxId(item)" class="form-check-label">{{item.text}}</label>
                    </div>
                </div>
            </div>
        </div>
        <div class="row row-height">
          <div class="col-md-3 col-height col-label">
            <label for="remark">{{ labels.parameters_label }}</label>
          </div>
          <div class="col-md-9 col-height">
            <input type="text" class="form-control input-sm input-item" id="parametersdialog" v-model="localData.parameters" maxlength="80" />
          </div>
        </div>
    </template>
    <template v-slot:footer>
      <button ref="okbuttondialogitem" id="okbuttondialogitem" class="btn btn-dark btn-sm" @click="okClick"><em class="fa fa-check fa-btn-icon"></em>{{ labels.ok_button }}</button>
      <button id="canceldialogbutton" class="btn btn-dark btn-sm" data-dismiss="modal"><em class="fa fa-close fa-btn-icon"></em>{{ labels.cancel_button }}</button>
    </template>
  </DialogForm>
</template>
<script>
import { ref, computed, watch } from 'vue';
import { useVuelidate } from '@vuelidate/core';
import { required, helpers } from '@vuelidate/validators';
import $ from "jquery";
import { disableControls }  from '@willsofts/will-app';
import DialogForm from './DialogForm.vue';

const defaultData = {
  progid: "",
  progtitle: "",
  parameters: "",
  permits: {},
};

export default {
  components: {
    DialogForm
  },
  props: {
    modes: Object,
    labels: Object,
    dataCategory: Object
  },
  emits: ["data-submit"],
  setup(props) {
    const localData = ref({ ...defaultData }); 
    const disabledKeyField = ref(false);
    const reqalert = ref(props.labels.empty_alert);
    const requiredMessage = () => {
      return helpers.withMessage(reqalert, required);
    }
    const validateRules = computed(() => { 
      return {
        progid: { required: requiredMessage() },
        progtitle: { required: requiredMessage() },
      } 
    });
    const submits = ref({submitCallback: undefined})
    const v$ = useVuelidate(validateRules, localData, { $lazy: true, $autoDirty: true });
    return { v$, localData, disabledKeyField, reqalert, submits };
  },
  created() {
    watch(this.$props, (newProps) => {      
      this.reqalert = newProps.labels.empty_alert;
      let sourceAry = [];
      let programs = newProps.dataCategory.tprog;
      programs.forEach((element) => {
        sourceAry.push({ label: element.id+" "+element.text, value: element.id, text: element.text });
      });
      $("#progiddialog").autocomplete("option","source",sourceAry);
    });
  },
  mounted() {
    this.$nextTick(() => {
      $("#item_dialog_layer").find(".modal-dialog").draggable();
      let sourceAry = [];
      let programs = this.$props.dataCategory.tprog;
      programs.forEach((element) => {
        sourceAry.push({ label: element.id+" "+element.text, value: element.id, text: element.text });
      }); 
      $("#progiddialog").autocomplete({
        delay: 500, 
        source: sourceAry,
        select : (event, ui) => {
          console.log("on select: ",ui.item);
          this.localData.progid = ui.item.value;
          this.localData.progtitle = ui.item.text;
        },
        open: function(){
          $(this).autocomplete('widget').css('z-index', 1090);
          return false;
        },	
      });
      $("#progiddialog").autocomplete("widget").addClass("autocomplete-fixed-height");
    });
  },
  methods: {
    getBoxId(item) { return "permitdialog"+item.id; },
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
        this.localData = {...newData, permits: permits};
      }
    },
    async okClick() {
      console.log("click: ok");
      disableControls($("#okbuttondialogitem"));
      let valid = await this.validateForm();
      if(!valid) return;
      this.submitRecord();
    },
    async validateForm(focusing=true) {
      const valid = await this.v$.$validate();
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
    showDialog() {
      $(this.$refs.dialogForm.$el).on("shown.bs.modal",() => { this.$refs.progiddialog.focus(); });
      $(this.$refs.dialogForm.$el).modal("show");
    },  
    hideDialog() {
      $(this.$refs.dialogForm.$el).modal("hide");
    },
    resetRecord() {
      this.reset(defaultData);
      this.v$.$reset();
      this.disabledKeyField = false;
    },
    submitRecord() {
      if(this.submits.submitCallback) {
        this.submits.submitCallback(this.localData);
      }
      this.$emit('data-submit',this.localData);
    },
    show(callback) {
      this.submits.submitCallback = callback;
      this.resetRecord();
      this.showDialog();
    },
  }
};
</script>
