<template>
	<v-container grid-list-md>
    <v-layout row wrap>
      <v-flex xs12 sm12 md12 class="mb-1">
        <v-card flat>
          <v-card-title class="title font-weight-light trainamodel white--text">
            <v-icon color="white" class="mr-3">school</v-icon>
            <span>Train a New Model</span>
          </v-card-title>
            <v-stepper v-model="step" @keydown.enter="step++">
              <v-stepper-header>
                <v-stepper-step :complete="step>1" step="1">Upload</v-stepper-step>
                <v-divider></v-divider>
                <v-stepper-step :complete="step>2" step="2">Target</v-stepper-step>
                <v-divider></v-divider>
                <v-stepper-step :complete="step>3" step="3">Training Columns</v-stepper-step>
                <v-divider></v-divider>
                <v-stepper-step :complete="step>4" step="4">Types</v-stepper-step>
                <v-divider></v-divider>
                <v-stepper-step :complete="step>5" step="5">Send</v-stepper-step>
              </v-stepper-header>

              <v-stepper-items>
                <v-stepper-content step="1">
                  <v-card flat class="mb-3 background">
                    <v-card-title class="subheading font-weight-bold">
                      <span>Upload Your Dataset</span>
                      <v-spacer></v-spacer>
                      <v-tooltip left color="trainamodel">
                        <v-icon slot="activator" color="#9d9d9d">help</v-icon>
                        <ul style="padding-left: 1.2em">
                          <li>.csv, .xlsx and .xls files are supported.</li>
                          <li>Missing values will be automatically filled later.</li>
                        </ul>
                      </v-tooltip>
                    </v-card-title>
                    <v-card-text>
                      <v-layout align-center>
                        <upload-btn v-if="!loaders.upload" class="px-0 white--text" color="trainamodel" :ripple="false" :fileChangedCallback="upload"></upload-btn>
                        <v-btn v-if="loaders.upload" :disabled="true" :loading="true">Upload</v-btn>
                        <span>{{ allInfos.fileName }}</span>
                      </v-layout>
                      <p class="mb-0 mt-2 error--text" v-if="allInfos.error != ''">{{ allInfos.error }}</p>
                    </v-card-text>
                  </v-card>
                  <v-btn class="trainamodel white--text ml-0" :disabled="!allInfos.valid || loaders.upload" @click="step++">Next</v-btn>
                </v-stepper-content>

                <v-stepper-content step="2">
                  <v-card flat class="mb-3 background">
                    <v-card-title class="subheading font-weight-bold">
                      <span>Select Your Target Column</span>
                      <v-spacer></v-spacer>
                      <v-tooltip left color="trainamodel">
                        <v-icon slot="activator" color="#9d9d9d">help</v-icon>
                        <ul style="padding-left: 1.2em">
                          <li>Target column should be binary<br>(churn or not churn).</li>
                        </ul>
                      </v-tooltip>
                    </v-card-title>
                    <v-card-text>
                      <v-select v-model="targetCol" :items="targetableCols" label="Target column"></v-select>
                      <span class="error--text" v-if="targetableCols.length == 0 && step == 2">There is no proper target column in your dataset. Please upload another one.</span>
                    </v-card-text>
                  </v-card>
                  <v-btn class="trainamodel white--text ml-0" @click="backTo(1)">Previous</v-btn>
                  <v-btn class="trainamodel white--text ml-0" :disabled="targetableCols.length==0" @click="step3">Next</v-btn>
                </v-stepper-content>

                <v-stepper-content step="3">
                  <v-card flat class="mb-3 background">
                    <v-card-title class="subheading font-weight-bold">
                      <span>Select Your Training Columns</span>
                      <v-spacer></v-spacer>
                      <v-tooltip left color="trainamodel">
                        <v-icon slot="activator" color="#9d9d9d">help</v-icon>
                        <ul style="padding-left: 1.2em">
                          <li>Categorical columns with more than 30 categories<br>cannot be selected due to improve server<br>performance and model accuracy.</li>
                        </ul>
                      </v-tooltip>
                    </v-card-title>
                    <v-card-text>
                      <v-select v-model="selectedTrainCols" :items="allTrainCols" item-text="name" label="Select" :menu-props="{ maxHeight: '400' }" return-object multiple></v-select>
                    </v-card-text>
                  </v-card>
                  <v-btn class="trainamodel white--text ml-0" @click="backTo(2)">Previous</v-btn>
                  <v-btn class="trainamodel white--text ml-0" :disabled="!colsSelected" @click="step++">Next</v-btn>
                </v-stepper-content>

                <v-stepper-content step="4">
                  <v-card flat class="mb-3 background">
                    <v-card-title class="subheading font-weight-bold">
                      <span>Configure Your Column Types</span>
                      <v-spacer></v-spacer>
                      <v-tooltip left color="trainamodel">
                        <v-icon slot="activator" color="#9d9d9d">help</v-icon>
                        <ul style="padding-left: 1.2em">
                          <li>This step contains only your training columns.</li>
                          <li>Columns with non-numeric values are selected as<br>categoric.</li>
                          <li>Columns with numeric values are selected as <br>numeric.</li>
                          <li>Please check your dataset to see if there are<br>categoric columns with numeric values.</li>
                          <li>Columns having more than 30 unique values cannot<br>be selected as categoric.</li>
                        </ul>
                      </v-tooltip>
                    </v-card-title>
                    <v-card-text>
                      <p class="mb-2 subheading" v-if="catList.length != 0">The following columns are automatically detected as categoric:</p>
                      <p class="mb-2 subheading" v-if="catList.length == 0">No columns automatically detected as categoric.</p>
                      <v-chip :class="'ml-0 mr-2 chip'+(col.cat == 1 || moreCatCols.includes(col))+' white--text'" small disabled :key="col.name" v-for="col in catList">{{ col.name }}</v-chip>
                      <p class="my-2 subheading" v-if="catList.length != 0 && catable.length != 0">If there are more categoric columns, please select from below.</p>
                      <p class="my-2 subheading" v-if="catList.length == 0 && catable.length != 0">If there are categoric columns, please select from below.</p>
                      <p class="my-2 subheading" v-if="catable.length == 0">There are no columns that can be selected as categoric.</p>
                      <v-select v-if="catable.length != 0" v-model="moreCatCols" :items="catable" item-text="name" label="Select" :menu-props="{ maxHeight: '400' }" return-object multiple></v-select>
                    </v-card-text>
                  </v-card>
                  <v-btn class="trainamodel white--text ml-0" @click="backTo(3)">Previous</v-btn>
                  <v-btn class="trainamodel white--text ml-0" @click="step++">Next</v-btn>
                </v-stepper-content>

                <v-stepper-content step="5">
                  <v-card flat class="mb-3 background">
                    <v-card-title class="subheading font-weight-bold">
                      <span>Send Your Preferences</span>
                      <v-spacer></v-spacer>
                      <v-tooltip left color="trainamodel">
                        <v-icon slot="activator" color="#9d9d9d">help</v-icon>
                        <ul style="padding-left: 1.2em">
                          <li>Your dataset will not be saved.</li>
                        </ul>
                      </v-tooltip>
                    </v-card-title>
                    <v-card-text>
                      <v-form ref="form" v-model="modelNameValid" onSubmit="return false;">
                        <v-text-field v-model="modelName" label="Enter your models name" required :rules="modelNameRules" @keydown.enter="sendUserPrefs"></v-text-field>
                      </v-form>
                      <p class="mb-0">
                        <span class="subheading font-weight-medium">Dataset: </span>
                        <span class="subheading">{{ allInfos.fileName }}</span>
                      </p>
                      <p class="mb-0">
                        <span class="subheading font-weight-medium">Training columns: </span>
                        <v-tooltip top :key="col.name" v-for="col in selectedTrainCols">
                          <v-chip slot="activator" :class="'ml-0 mr-2 chip'+(col.cat == 1 || moreCatCols.includes(col))+' white--text'" small disabled>{{ col.name }}</v-chip>
                          <span v-if="col.cat == 1 || moreCatCols.includes(col)">Categoric</span>
                          <span v-if="col.cat == 0 && !moreCatCols.includes(col)">Numeric</span>
                        </v-tooltip>
                      </p>
                      <p class="mb-0">
                        <span class="subheading font-weight-medium">Target column: </span>
                        <v-chip small disabled class="trainamodel white--text">{{ targetCol }}</v-chip>
                      </p>
                      <v-checkbox v-model="isCustomized" @change="isCustomized ? dialogs[2].show = true : dialogs[2].show = false" label="Train with custom parameters" :height="0" color="trainamodel"></v-checkbox>
                      <v-dialog v-model="dialogs[2].show" max-width="600px" persistent>
                        <custom-model v-if="step == 5"></custom-model>
                      </v-dialog>
                      <p class="mb-0" v-if="isCustomized && !dialogs[2].show && selectedParams.modelType != undefined">
                        <span class="subheading font-weight-medium">Selected Parameters: </span>
                        <span>{{ selectedParams.modelType }}</span>
                      </p>
                      <span class="error--text" v-if="sendError.show">{{ sendError.msg }}</span>
                    </v-card-text>
                  </v-card>
                  <v-btn class="trainamodel white--text ml-0" @click="backTo(4)">Previous</v-btn>
                  <v-btn class="trainamodel white--text ml-0" :loading="loaders.send" :disabled="loaders.send || !modelNameValid" @click="sendUserPrefs">Send</v-btn>
                </v-stepper-content>

                <v-stepper-content step="6">
                  <v-card flat class="mb-3 background">
                    <v-card-title class="subheading font-weight-bold">Success</v-card-title>
                    <v-card-text>
                      <p class="mb-0 subheading">Your model is being trained.</p>
                    </v-card-text>
                  </v-card>
                  <v-btn class="ml-0 trainamodel white--text" @click="backTo(1)">Train another model</v-btn>
                </v-stepper-content>
              </v-stepper-items>
            </v-stepper>
        </v-card>
      </v-flex>

      <v-flex xs12 sm6 md6 v-if="allInfos.valid">
        <v-card flat ripple color="datatable" @click.stop="dialogs[0].show = true" style="cursor: pointer">
          <v-card-title class="title font-weight-light white--text">
            <v-icon color="white" class="mr-3">table_chart</v-icon>
            <span>Data Table</span>
          </v-card-title>
          <v-dialog v-model="dialogs[0].show" max-width="1250px">
            <datatable :dataset="allInfos.dataset" :columns="allInfos.columns"></datatable>
          </v-dialog>
        </v-card>
      </v-flex>

      <v-flex xs12 sm6 md6 v-if="allInfos.valid">
        <v-card flat ripple color="charts" @click.stop="dialogs[1].show = true" style="cursor: pointer">
          <v-card-title class="title font-weight-light white--text">
            <v-icon color="white" class="mr-3">insert_chart</v-icon>
            <span>Charts</span>
          </v-card-title>
          <v-dialog v-model="dialogs[1].show" max-width="1250px">
            <charts :colInfos="allInfos.colInfos" :title="'Charts'"></charts>
          </v-dialog>
        </v-card>
      </v-flex>
    </v-layout>
	</v-container>
</template>

<script>
  import DataTable from './DataTable'
  import Charts from './Charts'
  import UploadButton from 'vuetify-upload-button';
  import { EventBus } from "../plugins/event-bus.js";
  import CustomModel from './CustomModel';

  export default {
    components: {
      'datatable': DataTable,
      'charts': Charts,
      'upload-btn': UploadButton,
      'custom-model': CustomModel
    },
    data:() => ({
      modelNameValid: false,
      modelNameRules: [
        v => !!v || 'Model name is required',
        v => v != null && v.length <= 20 && v.length >= 3 || 'Model name must be between than 3 and 20 characters'
      ],
      dialogs: [{ name: 'datatable', show: false }, { name:'charts', show: false }, { name:'custommodel', show: false }],
      sendError: { msg: '', show: false },
      loaders: {
        upload: false,
        send: false
      },
      allInfos: {
        error: '',
        valid: false,
        fileName: '',
        columns: [],
        dataset: [],
        colInfos: []
      },
      targetCol: '',
      step: 0,
      selectedTrainCols: [],
      moreCatCols: [],
      modelName: '',
      sent: false,
      isCustomized: false,
      selectedParams: {}
    }),
    created() {
      EventBus.$on('close', val => { this.dialogs[val].show = false; });
      EventBus.$on('changed', (params, isCustomized) => {
        this.selectedParams = params;
        this.isCustomized = isCustomized;
      });
    },
    computed: {
      targetableCols() {
        let columns = [];
        this.allInfos.colInfos.forEach(val => {
          if (val.values.length == 2)
            columns.push(val.name);
        });
        // eslint-disable-next-line
        if (columns.length != 0) this.targetCol = columns[columns.length-1];
        return columns
      },
      allTrainCols() {
        let col = this.targetCol;
        return this.allInfos.colInfos.filter(c => {
          return c.name != col && (c.cat == 0 || c.values.length <= 30)
        })
      },
      colsSelected() {
        return (this.selectedTrainCols.length > 0)
      },
      catList() {
        return this.selectedTrainCols.filter(c => { return c.cat == 1 })
      },
      numList() {
        return this.selectedTrainCols.filter(c => { return c.cat == 0 })
      },
      catable() {
        return this.numList.filter(c => { return c.values.length <= 30 })
      }
    },
    methods: {
      upload(file) {
        this.loaders.upload = true;
        this.allInfos.valid = false;
        this.$parse(file, this.$session.get('uid')).then(result => {
          this.loaders.upload = false;
          this.allInfos = result;
        });
      },
      step3() {
        this.allTrainCols.forEach(val => {
          if (!this.selectedTrainCols.includes(val))
            this.selectedTrainCols.push(val);
        });
        this.step++;
      },
      sendUserPrefs() {
        if (this.modelNameValid && !this.loaders.send) {
          this.loaders.send = true;
          let cats = [];
          let nums = [];
          let col = {};
          while ((col = this.moreCatCols.pop()) != null)
            col.cat = 1;
          this.catList.forEach(val => { cats.push(this.allInfos.colInfos.map(c => { return c.name; }).indexOf(val.name)); });
          this.numList.forEach(val => { nums.push(this.allInfos.colInfos.map(c => { return c.name; }).indexOf(val.name)); });
          let targetCol = this.allInfos.columns.indexOf(this.targetCol);
          let class1 = this.allInfos.dataset.filter(c => { return c[targetCol] == this.allInfos.colInfos[targetCol].values[0] });
          let class2 = this.allInfos.dataset.filter(c => { return c[targetCol] == this.allInfos.colInfos[targetCol].values[1] });
          let minority = Math.min(this.allInfos.colInfos[targetCol].counts[0], this.allInfos.colInfos[targetCol].counts[1]);
          class1.length = minority;
          class2.length = minority;
          let newDataset = class1.concat(class2);
          let trainParams = { modelname: this.modelName, dataset: newDataset, columns: this.allInfos.columns, target: targetCol, categoricalcolumns: cats, numericalcolumns: nums, uid: this.$session.get('uid') };
          Object.keys(this.selectedParams).forEach(key => {
            trainParams[key] = this.selectedParams[key];
          });
          this.$post('/train', trainParams).then(data => {
            this.loaders.send = false;
            if(data.info == 1) {
              this.step = 6;
              this.sent = true;
              this.sendError.show = false;
            } else if (data.info == 0) {
              this.sendError.msg = data.details;
              this.sendError.show = true;
            } else if (data.info == -1) {
              this.sendError.msg = 'Server error.';
              this.sendError.show = true;
            }
          });
        }
      },
      backTo(step) {
        this.step = step;
        switch(step) {
          case 1:
            this.allInfos = {
              error: '',
              valid: false,
              fileName: '',
              columns: [],
              dataset: [],
              colInfos: []
            };
            this.sent = false;
            this.selectedTrainCols = [];
            this.moreCatCols = [];
            this.modelName = '';
            this.modelNameValid = false;
            this.$refs.form.reset();
            this.isCustomized = false;
            this.selectedParams = {};
            break;
          case 2:
            this.selectedTrainCols = [];
            break;
          case 3:
            this.moreCatCols = [];
            break;
          case 4:
            this.modelName = '';
            this.modelNameValid = false;
            this.$refs.form.reset();
            this.isCustomized = false;
            this.selectedParams = {};
            this.sendError.show = false;
        }
      }
    }
  }
</script>