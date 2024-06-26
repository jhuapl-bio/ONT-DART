<!-- 
  Copyright 2022, The Johns Hopkins University Applied Physics Laboratory LLC
  All rights reserved.
  Distributed under the terms of the BSD 3-Clause License.

  NO WARRANTY, NO LIABILITY. THIS MATERIAL IS PROVIDED “AS IS.” JHU/APL MAKES NO
  REPRESENTATION OR WARRANTY WITH RESPECT TO THE PERFORMANCE OF THE MATERIALS, INCLUDING
  THEIR SAFETY, EFFECTIVENESS, OR COMMERCIAL VIABILITY, AND DISCLAIMS ALL WARRANTIES IN
  THE MATERIAL, WHETHER EXPRESS OR IMPLIED, INCLUDING (BUT NOT LIMITED TO) ANY AND ALL
  IMPLIED WARRANTIES OF PERFORMANCE, MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE,
  AND NON-INFRINGEMENT OF INTELLECTUAL PROPERTY OR OTHER THIRD PARTY RIGHTS. ANY USER OF
  THE MATERIAL ASSUMES THE ENTIRE RISK AND LIABILITY FOR USING THE MATERIAL. IN NO EVENT
  SHALL JHU/APL BE LIABLE TO ANY USER OF THE MATERIAL FOR ANY ACTUAL, INDIRECT,
  CONSEQUENTIAL, SPECIAL OR OTHER DAMAGES ARISING FROM THE USE OF, OR INABILITY TO USE,
  THE MATERIAL, INCLUDING, BUT NOT LIMITED TO, ANY DAMAGES FOR LOST PROFITS.
 -->
 <template>
  <v-card :height="height" class="d-flex flex-column">
    <v-card-title>Analysis Form</v-card-title>
    <v-card-text>
      <v-form
        ref="form"
        v-model="valid"
        lazy-validation
      >
        <v-tooltip bottom>
          <template v-slot:activator="{ on, attrs }">
            <v-text-field
              v-model="inputDirectory"
              v-bind="attrs"
              v-on="on"
              label="Input Directory"
              type="text"
              :rules="[rules.required]"
              required
            ></v-text-field>
          </template>
          <span>
            The input directory must be the full path to the 'fastq_pass' subfolder of an ONT sequencing run,
            and have sub-folders named for each demultiplexed (demux) barcode (this is default for demux by MinKNOW).
          </span>
        </v-tooltip>

        <v-tooltip bottom>
          <template v-slot:activator="{ on, attrs }">
            <v-text-field
              v-model="refDirectory"
              v-bind="attrs"
              v-on="on"
              label="Reference Database"
              type="text"
              :rules="[rules.required]"
              required
            ></v-text-field>
          </template>
          <span>
            The reference directory must be the full path to the Amplicon database and prefix of the database created by blastn.
            For example /home/database/Amplicon_blastdb/Amplicon_set. 
          </span>
        </v-tooltip>

        <v-tooltip bottom>
          <template v-slot:activator="{ on, attrs }">
            <v-text-field
              v-model="orgFile"
              v-bind="attrs"
              v-on="on"
              label="Organism File"
              type="text"
              :rules="[rules.required]"
              required
            ></v-text-field>
          </template>
          <span>
            The path to the organisms.sh file structure with all organism_IDs and names defined for parsing.
          </span>
        </v-tooltip>

        <v-tooltip bottom>
          <template v-slot:activator="{ on, attrs }">
            <v-text-field
              v-model="barcodes"
              v-bind="attrs"
              v-on="on"
              label="NTC Barcodes"
              type="text"
              :rules="[rules.required]"
              required
            ></v-text-field>
          </template>
          <span>
            Additionally, one or more of the barcodes is reserved for a NTC (nontemplate control).
            This is used for the final threshold for calling a positive amplicon (T3).
          </span>
        </v-tooltip>

        <v-tooltip bottom>
          <template v-slot:activator="{ on, attrs }">
            <v-text-field
              v-model="analysisInterval"
              v-bind="attrs"
              v-on="on"
              label="Analysis Interval (in seconds)"
              type="number"
              :rules="[rules.required]"
              required
            ></v-text-field>
          </template>
          <span>
            The analysis interval is set to running the analysis on the input directory every 60 seconds.
            A maximum of 10 minutes is set internally, and will override any value larger than 600.
            This is to keep analysis processing time within a reasonable range.
            Note that reads processed in the previous interval are not processed in the next,
            and their counts are carried over to the latest results files.
          </span>
        </v-tooltip>

        <v-tooltip bottom>
          <template v-slot:activator="{ on, attrs }">
            <v-text-field
              v-model="threads"
              v-bind="attrs"
              v-on="on"
              label="Number of Threads"
              type="number"
              :rules="[rules.required]"
              required
            ></v-text-field>
          </template>
          <span>
            The default thread count is set to 1, and should only be increased if you know your system well.
            It is recommended that if sequencing, basecalling, and demultiplexing are running concurrently with this application,
            that the thread count be set no greater than half the total threads available on the system.
          </span>
        </v-tooltip>

        <!-- New refThresh field -->
        <v-tooltip bottom>
          <template v-slot:activator="{ on, attrs }">
            <v-text-field
              v-model="refThresh"
              v-bind="attrs"
              v-on="on"
              label="Reference Alignment and Identity Threshold"
              type="text"
              :rules="[rules.required]"
              required
            ></v-text-field>
          </template>
          <span>
            The reference alignment and identity threshold for considering a hit,
            for example '90' means the amplicon read alignment must cover >= 90%  of the reference length and be >=90% the identity.
          </span>
        </v-tooltip>
        
      </v-form>
    </v-card-text>
    <v-card-actions class="justify-end">
      <v-btn
        color="success"
        :disabled="!valid"
        class="ml-6"
        @click="startAnalysis"
      >
        Start
      </v-btn>
      <v-btn
        color="error"
        :disabled="!stopvalid"
        class="ml-2"
        @click="stopAnalysis"
      >
        Stop
      </v-btn>
    </v-card-actions>
  </v-card>
</template>

<script>
import AnalysisService from '@/services/analysis.service'

export default {
    name: 'AnalysisForm',
    props: {
      height: String
    },
    data: () => ({
        valid: true,
        stopvalid: false,
        inputDirectory: '/app/DATA/fastq_pass/',
        refDirectory: '/app/DATA/Amplicon_blastdb/Amplicon_set',
        orgFile: '/app/DATA/organisms.sh',
        barcodes: 'barcode10,barcode11,barcode12',
        analysisInterval: '10',
        threads: '1',
        refThresh: '90',
        rules: {
          required: value => !!value && value.trim() !== '' || 'This field is required'
        }
    }),
    methods: {
        startAnalysis() {
          if (!this.$refs.form.validate()) {
            return
          }

          const data = {
            inputDirectory: this.inputDirectory,
            refDirectory: this.refDirectory,
            orgFile: this.orgFile,
            barcodeList: this.barcodes,
            analysisInterval: this.analysisInterval,
            numThreads: this.threads,
            refThresh: this.refThresh
          }

          AnalysisService.startAnalysis(data)
            .then(this.handleSuccess)
            .catch(this.handleError)
        },
        stopAnalysis() {
          AnalysisService.stopAnalysis()
            .then(this.handleSuccess)
            .catch(this.handleError)
        },
        handleSuccess(response) {
            this.valid = false
            this.stopvalid = true
            const data = response.data
            this.$emit('response', { message: data.message, type: 'success' })
        },
        handleError(error) {
            const response = error.response
            const message = response.status !== 500 ? response.data.message : 'An error occurred with the server.'
            this.$emit('response', { message: message, type: 'error' })
        }
    }
}
</script>

<style scoped>
.v-tooltip__content {
  background-color: DimGray !important;
  opacity: 1 !important;
  transition: none !important;
  max-width: 350px !important
}
</style>


