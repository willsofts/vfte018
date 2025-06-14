<template>
  <DialogForm ref="dialogForm">
    <template v-slot:header>
      <h4 class="modal-title" v-if="insertMode">{{ labels.title_new }}</h4>
      <h4 class="modal-title" v-if="updateMode">{{ labels.title_edit }}</h4>
    </template>
    <template v-slot:default>
        <div class="row row-height">
          <div class="col-height col-md-11">
            <label for="tenantname">{{ labels.tenantname_label }}</label>
            <div class="input-group has-validation" :class="{'has-error': v$.tenantname.$error}">
              <input type="text" ref="tenantname" v-model="localData.tenantname" id="tenantname" class="form-control input-md" /> 
              <label class="required">*</label>
            </div>
            <span v-if="v$.tenantname.$error" class="has-error">{{ v$.tenantname.$errors[0].$message }}</span>
          </div>
        </div>
        <div class="row row-height copy-layer" v-show="displayCopy">
          <div class="col-height col-md-11">
            <label for="tenantid" class="control-label">{{ labels.tenantid_label }}</label>
            <div class="input-group">
              <input type="text" ref="tenantid" id="tenantid" v-model="localData.tenantid" class="form-control input-md" readonly />
              <a href="javascript:void(0);" class="input-group-addon input-group-append input-group-text" @click="copyClient" tabindex="-1" title="Copy"><i class="fa fa-copy" aria-hidden="true"></i></a>
            </div>
					</div>
        </div>
        <div class="row row-height hidden-layer copy-layer" v-show="displayCopy">
          <div class="col-height col-md-11">
            <label for="publickeys" id="publickeys_label" class="control-label">{{ labels.publickeys_label }}</label>
            <div class="input-group">
              <textarea ref="publickeys" id="publickeys" class="form-control input-md" rows="6" v-model="localData.publickeys" readonly></textarea>
              <div class="input-group-required input-group-addon input-group-append">
                <table>
                  <tr><td valign="top">
                  <a href="javascript:void(0);" class="input-group-text key-link" @click="copyPublicKey" tabindex="-1" title="Copy"><i class="fa fa-copy" aria-hidden="true"></i></a>
                  <a href="javascript:void(0);" class="input-group-text key-link" @click="downloadPublicKey" tabindex="-1" title="Download"><i class="fa fa-download" aria-hidden="true"></i></a>
                  </td></tr>
                </table>
              </div>
            </div>
					</div>
        </div>
        <div class="row row-height hidden-layer" v-show="updateMode">
          <div class="col-height col-md-4">
            <label>{{ labels.inactive_label }}</label>
            <select ref="inactive" v-model="localData.inactive" class="form-control input-md">
              <option v-for="item in dataCategory.tkactive" :key="item.id" :value="item.id">{{item.text}}</option>
            </select>
          </div>
        </div>
    </template>
    <template v-slot:footer>
      <button ref="savebutton" id="savebutton" class="btn btn-dark btn-sm" @click="saveClick" v-show="displayMode"><em class="fa fa-save fa-btn-icon"></em>{{ labels.save_button }}</button>
      <button ref="updatebutton" id="updatebutton" class="btn btn-dark btn-sm" @click="updateClick" v-if="updateMode"><em class="fa fa-save fa-btn-icon"></em>{{ labels.update_button }}</button>
      <button id="canceldialogbutton" class="btn btn-dark btn-sm" data-dismiss="modal"><em class="fa fa-close fa-btn-icon"></em>{{ labels.cancel_button }}</button>
    </template>
  </DialogForm>
</template>
<script>
import { ref, computed, watch } from 'vue';
import { useVuelidate } from '@vuelidate/core';
import { required, helpers } from '@vuelidate/validators';
import $ from "jquery";
import { DEFAULT_CONTENT_TYPE, getApiUrl, disableControls }  from '@willsofts/will-app';
import { startWaiting, stopWaiting, submitFailure, detectErrorResponse }  from '@willsofts/will-app';
import { confirmUpdate, confirmSave, confirmDelete, successbox, serializeParameters } from '@willsofts/will-app';
import DialogForm from './DialogForm.vue';

const APP_URL = "/api/sfte018";
const defaultData = {
  tenantid: "",
  tenantname: "",
  publickeys: "",
  inactive: "0",
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
  emits: ["data-saved","data-updated","data-deleted"],
  setup(props) {
    const mode = ref({action: "new", ...props.modes});
    const localData = ref({ ...defaultData }); 
    const disabledKeyField = ref(false);    
    const reqalert = ref(props.labels.empty_alert);
    const requiredMessage = () => {
      return helpers.withMessage(reqalert, required);
    }
    const validateRules = computed(() => { 
      return {
        tenantname: { required: requiredMessage() },
      } 
    });
    const v$ = useVuelidate(validateRules, localData, { $lazy: true, $autoDirty: true });
    const showButton = ref(true);
    const displayCopy = ref(false);
    return { mode, v$, localData, disabledKeyField, reqalert, displayCopy, showButton };
  },
  created() {
    watch(this.$props, (newProps) => {      
      this.reqalert = newProps.labels.empty_alert;
    });
  },
  computed: {
    insertMode() {
      return this.mode.action=="insert" || this.mode.action=="new";
    },
    updateMode() {
      return this.mode.action=="update" || this.mode.action=="edit";
    },
    displayMode() {
      return this.showButton && (this.mode.action=="insert" || this.mode.action=="new");
    }
  },
  mounted() {
    this.$nextTick(function () {
      $("#modaldialog_layer").find(".modal-dialog").draggable();
    });
  },
  methods: {
    reset(newData,newMode) {
      if(newData) this.localData = {...newData};
      if(newMode) this.mode = {...newMode};
    },
    async saveClick() {
      console.log("click: save");
      disableControls($("#savebutton"));
      let valid = await this.validateForm();
      if(!valid) return;
      this.startSaveRecord();
    },
    async updateClick() {
      console.log("click: update");
      disableControls($("#updatebutton"));
      let valid = await this.validateForm();
      if(!valid) return;
      this.startUpdateRecord();
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
    showDialog(callback) {
      if(callback) $(this.$refs.dialogForm.$el).on("shown.bs.modal",callback);
      $(this.$refs.dialogForm.$el).modal("show");
    },  
    hideDialog() {
      $(this.$refs.dialogForm.$el).modal("hide");
    },
    resetRecord() {
      //reset to default data 
      this.reset(defaultData,{action:"insert"});
      //reset validator
      this.v$.$reset();
      //enable key field
      this.disabledKeyField = false;
      this.showButton = true;
      this.displayCopy = false;
    },
    startInsertRecord() {
      this.resetRecord();
      this.showDialog(() => { this.$refs.tenantname.focus(); });
    },
    startSaveRecord() {
      confirmSave(() => {
        let data = {tenantid: this.localData.tenantid, tenantname: this.localData.tenantname, inactive: this.localData.inactive};
        this.saveRecord(data);
      });
    },
    startUpdateRecord() {
      confirmUpdate(() => { 
        this.updateRecord(this.localData);
      });
    },
    startDeleteRecord(dataKeys) {
      confirmDelete(Object.values(dataKeys),() => {
        this.deleteRecord(dataKeys);
      });
    },
    saveRecord(dataRecord) {
        let jsondata = {ajax: true};
        let formdata = serializeParameters(jsondata,dataRecord);
        startWaiting();
        $.ajax({
          url: getApiUrl()+APP_URL+"/insert",
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
            if(data.body.dataset) {
              this.reset(data.body.dataset,{action:"insert"});
              this.v$.$reset();
            }
            successbox(() => {
              //reset data for new record insert
              //this.resetRecord();
              //setTimeout(() => { this.$refs.tenantname.focus(); },100);
              this.showButton = false;
              this.displayCopy = true;
              this.$emit('data-saved',dataRecord,data);
            });
          }
      });
    },
    updateRecord(dataRecord) {
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
              this.hideDialog();
              this.$emit('data-updated',dataRecord,data);
            });
          }
      });
    },
    retrieveRecord(dataKeys) {
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
          if(data.body.dataset) {
            this.reset(data.body.dataset,{action:"edit"});
            this.v$.$reset();
            this.disabledKeyField = true;
            this.showButton = true;
            this.displayCopy = true;
            this.showDialog();
          }
        }
      });	
    },
    deleteRecord(dataKeys) {
      let jsondata = {ajax: true};
      let formdata = serializeParameters(jsondata,dataKeys);
      startWaiting();
      $.ajax({
        url: getApiUrl()+APP_URL+"/remove",
        data: formdata.jsondata,
        headers : formdata.headers,
        type: "POST",
        dataType: "json",
        contentType: DEFAULT_CONTENT_TYPE,
        error : function(transport,status,errorThrown) {
          console.error("deleteRecord: error: status",status,"errorThrown",errorThrown);
          submitFailure(transport,status,errorThrown);
        },
        success: (data) => {
          stopWaiting();
          console.log("deleteRecord: success",data);
          if(detectErrorResponse(data)) return;
          successbox(() => {
            this.$emit('data-deleted',dataKeys,data);
          });
        }
      });	
    },
    async copyClient() {
      try {
        let copyText = document.getElementById("tenantid");
        copyText.select();
        await navigator.clipboard.writeText(copyText.value);
      } catch(ex) { console.error(ex); }
    },
    async copyPublicKey() {
      try {
        let copyText = document.getElementById("publickeys");
        copyText.select();
        await navigator.clipboard.writeText(copyText.value);
      } catch(ex) { console.error(ex); }
    },
    downloadPublicKey() {
      this.createDownloader(this.localData.publickeys,this.localData.tenantid+".pem");
    },
    createDownloader(data, filename) {
      let a = window.document.createElement('a');
      a.href = window.URL.createObjectURL(new Blob([data], { type: 'text/html' }));
      a.download = filename;
      document.body.appendChild(a);
      a.click();
      document.body.removeChild(a);
    },
  }
};
</script>
