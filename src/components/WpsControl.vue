<template>
  <section>
    <label for="wpsprocess">
      <b>Seleziona un processo</b>
    </label>
    <select id="wpsprocess" v-model="current_process">
      <option disabled value="">Seleziona un processo</option>
      <option v-for="process in processes" :key="process.title" :value="process.title">{{ process.title }}</option>
    </select>
    <section v-if="current_process">
      <section class="process_abstract">
        <h3>Abstract</h3>
        <span>{{process_data.abstract}}</span>
      </section>
      <section class="process_inputs">
        <h2>Inputs</h2>
          <template v-for="input in process_data.inputs">
            <label :for="input.id">{{input.label}}</label>
            <input :id="input.id" :type="input.type" :value="input.value">
          </template>
      </section>
      <div class="process_outputs">
        <h2>Outputs</h2>
        <template v-for="output in process_data.outputs">
          <label :for="output.id">{{output.label}}</label>
          <input :id="output.id" :type="output.type">
        </template>
      </div>
      <section>
        <button>Run</button>
      </section>
    </section>
  </section>
</template>

<script>
  import jQuery from 'jquery'
  window.jQuery = window.$ = jQuery;
  export default {
    name: "WpsControl",
    data(){
      return {
        processes: [],
        current_process: undefined,
        process_data : {
          abstract: null,
          inputs:[],
          outputs:[]
        }
      }
    },
    async created() {
      this.wpsService = new WpsService({
        url: '/src/fakedata/wps/getcapabilities.xml',
        version: "1.0.0"
      });
      this.wpsService.getCapabilities_GET(response => {
        try {
          const processes = response.capabilities && response.capabilities.processes || [];
          this.processes = [processes[0], processes[1], processes[3]];
        } catch (err) {
          console.log(err)
        }
      });
    },
    methods: {
      createProcessInputType(input) {
        const inputObject = {
          type: null,
          dataType: null,
          value: null,
          options: {}
        };
        if (input.complexData) {
          inputObject.type = 'file';
          inputObject.options.mimetype = input.complexData.formats[0].mimeType;
        } else if (input.literalData) {
          inputObject.type = input.literalData.literalDataDomains[0].dataType.type;
          inputObject.dataType = inputObject.type;
          inputObject.options = null;
        } else if (input.boundingBoxData) {
          inputObject.value = {
            llx: null,
            lly: null,
            upx: null,
            upy: null
          };
          inputObject.type = 'bbox';
          inputObject.epsg = input.boundingBoxData.supportedCRSs[0];
          inputObject.options.epsg = input.boundingBoxData.supportedCRSs;
        }
        return inputObject;
      },
      buildFormFromDescribeProcessResponse({abstractValue, inputs, outputs}={}) {
        inputs = inputs.map((input) => {
          const inputType = this.createProcessInputType(input);
          return {
            id: input.identifier,
            label: input.title,
            ...inputType
          }
        });
        outputs = outputs.map((output)=> {
          return {
            id: output.identifier,
            label: output.title,
            sublabel: output.abstractValue || '',
            value: '',
            execute: output
          }
        });

        return {
          abstract: abstractValue,
          inputs,
          outputs
        }
      }
    },
    watch: {
      'current_process'(process){
        if (process) {
          const url = `/src/fakedata/wps/${process}.xml`;
          this.wpsService.setUrl(url);
          this.wpsService.describeProcess_GET(response =>{
            const describeProcessResponse = response.processOffering && response.processOffering.process;
            const {abstract, inputs, outputs}  = this.buildFormFromDescribeProcessResponse(describeProcessResponse);
            this.process_data.abstract = abstract;
            this.process_data.inputs = inputs;
            this.process_data.outputs = outputs;
            console.log(outputs)
          }, process);
        }
      }
    }
  }
</script>

<style scoped>
  input, button {
    width: 90%;
  }
</style>