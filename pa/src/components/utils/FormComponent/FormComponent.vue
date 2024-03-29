<template>
  <div>
    <b-form
      v-if="form"
      @reset="onReset"
      @submit.prevent="onSubmitFunction"
    >
      <b-form-group
        v-for="(fieldInfo, fieldName) in form.fields"
        :id="'input-group-' + fieldName"
        :key="fieldName"
        :class="fieldInfo.classes"
        :label="fieldInfo.label"
        :label-for="'input-' + fieldName"
      >
        <b-form-input
          v-if="fieldInfo.type === 'hidden'"
          :id="'input-' + fieldName"
          v-model="fieldInfo.data"
          :placeholder="'Enter ' + fieldInfo.label"
          :state="fieldInfo.valid"
          :type="'number'"
          :required="fieldInfo.required"
        />
        <b-form-input
          v-else-if="fieldInfo.type !== 'choice'"
          :id="'input-' + fieldName"
          v-model="fieldInfo.data"
          :placeholder="'Enter ' + fieldInfo.label"
          :state="fieldInfo.valid"
          :type="fieldInfo.type"
          :required="fieldInfo.required"
        />
        <b-form-select
          v-if="fieldInfo.type === 'choice'"
          :id="'input' + fieldName"
          v-model="fieldInfo.data"
          :options="fieldInfo.choices"
          :placeholder="'Enter' + fieldInfo.label"
          :state="fieldInfo.valid"
          :required="fieldInfo.required"
        />

        <b-form-invalid-feedback
          v-for="error in fieldInfo.errors"
          :id="'input-' + fieldName"
          :key="error"
        >
          {{ error }}
        </b-form-invalid-feedback>
      </b-form-group>

      <div
        v-if="!embedded"
      >
        <b-button
          class="m-1"
          type="submit"
          variant="primary"
        >
          Submit
        </b-button>
        <b-button
          class="m-1"
          type="reset"
          variant="danger"
        >
          Reset
        </b-button>
      </div>
    </b-form>
  </div>
</template>
<script>
import {Form} from "./form";
import {errorMsg, successCreateMsg, successUpdateMsg} from "../../../utils/msgHelpers";
import {RequestMethods} from "../../../utils/requestMethods";
import {prepareRequestData} from "./prepareRequestData";
import {parseOptions} from "./parseOptions";
import {getFinApi} from "../../../api/api";

export default {
  name: 'FormComponent',
  props: {
    embedded: {
      type: Boolean,
      default: () => false
    },
    entity: {
      type: Object,
      default: () => null
    },
    entityName: {
      type: String,
      default: () => 'Unknown Entity'
    },
    fieldsClasses: {
      type: Object,
      default: () => null
    },
    method: {
      type: String,
      required: true
    },
    onSubmit: {
      type: Function,
      default: () => undefined
    },
    propForm: {
      type: Object,
      default: () => null
    },
    requestUrl: {
      type: String,
      required: true
    }
  },
  data: function () {
    return {
      finApi: getFinApi(),
      form: null,
      onSubmitFunction: this.defaultOnSubmit,
    }
  },
  watch: {
    propForm: {
      deep: true,
      // eslint-disable-next-line no-unused-vars
      handler: function (newVal, oldVal) {
        this.form = newVal
      }
    }
  },
  created: function () {
    if (this.onSubmit() !== undefined) {
      this.onSubmitFunction = this.onSubmit
    }

    if (this.propForm) {
      this.form = this.propForm
    } else {
      let that = this;
      this.finApi
        .options(this.requestUrl)
        .then(response => {
          if (response.data) {
            let fieldsInfo = parseOptions(response.data['actions'][that.method])
            for (let field in this.fieldsClasses) {
              fieldsInfo[field].classes = this.fieldsClasses[field]
            }
            that.form = new Form(fieldsInfo, that.entity)
          }
        })
    }

    this.errorMsg = errorMsg
    this.successCreateMsg = successCreateMsg
    this.successUpdateMsg = successUpdateMsg

  },
  methods: {
    defaultOnSubmit: function () {
      this.form.validate()

      if (this.form.valid) {
        let {creationData, paramsQuery} = prepareRequestData(this.form)

        if (this.method === RequestMethods.PUT) {
          let updateUrl = this.requestUrl
          this.finApi
            .put(updateUrl, creationData, {params: paramsQuery})
            .then(response => {
              if (response.status === 200) {
                this.successUpdateMsg(this.entityName)
              } else {
                throw 'Response status is not supported'
              }
            })
            .catch(errorResponse => {
              this.errorMsg(errorResponse, this.form)
            })
        } else if (this.method === RequestMethods.POST) {
          this.finApi
            .post(this.requestUrl, creationData, {params: paramsQuery})
            .then(response => {
              if (response.status === 201) {
                this.successCreateMsg(this.entityName)
              } else {
                throw 'Response status is not supported'
              }
            })
            .catch(errorResponse => {
              this.errorMsg(errorResponse, this.form)
            })
        } else {
          throw Error('Unsupported method type')
        }
      } else {
        this.$bvToast.toast('Form is invalid')
      }
    },
    onReset: function (event) {
      this.event = event;
    },
  }
}
</script>
