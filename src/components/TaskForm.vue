<template>
  <DialogForm ref="dialogForm" id="task_dialog_layer">
    <template v-slot:header>
      <h4 class="modal-title">{{ labels.task_dialog_title }}</h4>
    </template>
    <template v-slot:default>
        <div class="row row-height">
          <div class="col-md-2 col-height col-label">
            <label for="progtitletask">{{ labels.progtitle_label }}</label>
          </div>
          <div class="col-md-8 col-height">
            <div class="input-group has-validation" :class="{'has-error': v$.progtitle.$error}">
              <input ref="progtitletask" type="text" v-model="localData.progtitle" id="progtitletask" class="form-control input-md" maxlength="100" />
            </div>
          </div>
        </div>
    </template>
    <template v-slot:footer>
      <button ref="okbuttondialogtask" id="okbuttondialogtask" class="btn btn-dark btn-sm" @click="okClick"><em class="fa fa-check fa-btn-icon"></em>{{ labels.ok_button }}</button>
      <button class="btn btn-dark btn-sm" data-dismiss="modal"><em class="fa fa-close fa-btn-icon"></em>{{ labels.cancel_button }}</button>
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
  progtitle: "",
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
    const reqalert = ref(props.labels.empty_alert);
    const requiredMessage = () => {
      return helpers.withMessage(reqalert, required);
    }
    const validateRules = computed(() => { 
      return {
        progtitle: { required: requiredMessage() },
      } 
    });
    const submits = ref({submitCallback: undefined})
    const v$ = useVuelidate(validateRules, localData, { $lazy: true, $autoDirty: true });
    return { v$, localData, reqalert, submits };
  },
  created() {
    watch(this.$props, (newProps) => {      
      this.reqalert = newProps.labels.empty_alert;
    });
  },
  mounted() {
    this.$nextTick(function () {
      $("#task_dialog_layer").find(".modal-dialog").draggable();
    });
  },
  methods: {
    reset(newData) {
      if(newData) this.localData = {...newData};
    },
    async okClick() {
      console.log("click: ok");
      disableControls($("#okbuttondialogtask"));
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
      $(this.$refs.dialogForm.$el).on("shown.bs.modal",() => { this.$refs.progtitletask.focus(); });
      $(this.$refs.dialogForm.$el).modal("show");
    },  
    hideDialog() {
      $(this.$refs.dialogForm.$el).modal("hide");
    },
    resetRecord() {
      this.reset(defaultData);
      this.v$.$reset();
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
