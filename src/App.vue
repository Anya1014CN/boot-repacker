<template>
  <v-app>
    <v-content>
      <v-container class="fill-height" fluid>
        <v-row align="center" justify="center">
          <v-col cols="12" sm="10" md="8">
            <header>
              <div class="langs">
                <v-select
                  solo
                  dense
                  flat
                  v-model="$root.$i18n.locale"
                  v-bind:items="langs"
                  prepend-inner-icon="mdi-web"
                />
              </div>
              <h1 v-html="$t('title')" />
            </header>

            <v-expansion-panels accordion mandatory v-model="panel" v-bind:readonly="!file">
              <v-expansion-panel>
                <v-expansion-panel-header>
                  <strong v-if="file == null">{{ $t('step1:title') }}</strong>
                  <strong v-else>{{ $t('step1:image') }}{{file.name}}</strong>
                </v-expansion-panel-header>
                <v-expansion-panel-content>
                  <FileDrop v-on:change="handleFile" />
                </v-expansion-panel-content>
              </v-expansion-panel>
              <v-expansion-panel>
                <v-expansion-panel-header><strong>{{ $t('step2:customize') }}</strong></v-expansion-panel-header>
                <v-expansion-panel-content>
                  <Form
                    v-bind:initial="initial"
                    v-bind:values="values"
                    v-on:change="handleChange"
                    v-on:replace="handleReplace"
                    v-on:remove="handleRemove"
                    v-on:export="handleExport"
                  />
                  <div class="d-flex">
                    <v-btn color="primary" text v-on:click="handleResetAll"><v-icon left>mdi-restore</v-icon>{{ $t('step2:reset') }}</v-btn>
                    <v-btn color="primary" text class="ml-auto" style="margin-right: 1em;" v-on:click="panel = 0">{{ $t('step2:back') }}</v-btn>
                    <v-btn color="primary" depressed v-on:click="panel = 2">{{ $t('step2:next') }}</v-btn>
                  </div>
                </v-expansion-panel-content>
              </v-expansion-panel>
              <v-expansion-panel>
                <v-expansion-panel-header><strong>{{ $t('step3:export') }}</strong></v-expansion-panel-header>
                <v-expansion-panel-content>
                  <div class="d-flex flex-column justify-center align-center" style="padding-top: 40px; padding-bottom: 40px;">
                    <v-btn color="primary" depressed large v-on:click="handleBuild">
                      <v-icon left>mdi-download</v-icon>{{ $t('step3:download') }}
                    </v-btn>
                  </div>
                </v-expansion-panel-content>
              </v-expansion-panel>
            </v-expansion-panels>

            <footer>
              Copyright &copy; 2020 MoKee Open Source Project,Modified by Anya1014 | <a href="https://github.com/Anya1014CN/boot-repacker">Github</a>
            </footer>
          </v-col>
        </v-row>
      </v-container>
    </v-content>
  </v-app>
</template>

<script>
import { saveAs } from 'file-saver';
import { debounce } from 'throttle-debounce';

import FileDrop from './components/FileDrop';
import Form from './components/Form';

import readFile from './utils/readFile';
import parseImage from './utils/parseImage';
import exportImage from './utils/exportImage'
import buildImage from './utils/buildImage';
import calculateImageId from './utils/calculateImageId';
import calculatePosition from './utils/calculatePosition';

import messages from './i18n';

const langs = Object.keys(messages)
  .sort()
  .map(lang => ({
    text: messages[lang].name,
    value: lang,
  }));

export default {
  name: 'App',
  components: {
    FileDrop,
    Form,
  },
  data() {
    return {
      langs: langs,
      panel: 0,
      file: null,
      blob: null,
      initial: {},
      values: {},
    };
  },
  methods: {
    async handleFile(file) {
      this.file = file;
      if (!file) return;
      try {
        this.blob = await readFile(file);
        this.initial = parseImage(this.blob);
        this.values = { ...this.initial };
        this.panel = 1;
      } catch (e) {
        console.error('Failed parsing image', e);
        this.file = null;
      }
    },
    handleChange(key, value) {
      this.values = { ...this.values, [key]: value };
    },
    handleResetAll() {
      this.values = { ...this.initial };
    },
    updateRecoveryDtboOffset() {
      const { offset, size } = calculatePosition('recovery_dtbo', this.values);
      this.values = { ...this.values, recovery_dtbo_offset: (size > 0 ? offset : 0) };
    },
    updateImageId: debounce(200, async function () {
      const img_id = await calculateImageId(this.blob, this.values);
      this.values = { ...this.values, img_id: img_id.toString('hex') };
    }),
    handleReplace(part, file) {
      const size = `${part}_size`;
      if (file) {
        this.values = {
          ...this.values,
          [part]: file,
          [size]: file.size,
        };
      } else {
        this.values = {
          ...this.values,
          [part]: this.initial[part],
          [size]: this.initial[size],
        };
      }
      this.updateRecoveryDtboOffset();
      this.updateImageId();
    },
    handleRemove(part) {
      const size = `${part}_size`;
      this.values = {
        ...this.values,
        [part]: null,
        [size]: 0,
      };
      this.updateRecoveryDtboOffset();
      this.updateImageId();
    },
    handleExport(part, name) {
      const image = exportImage(this.blob, this.initial, part);
      const blob = new Blob([ image ]);
      saveAs(blob, name);
    },
    async handleBuild() {
      const image = await buildImage(this.blob, this.values);
      const blob = new Blob([ image ]);
      saveAs(blob, `${this.file.name}-rebuilt.img`);
    },
  },
};
</script>

<style>
header {
  text-align: center;
  margin: 0 0 20px;
}

header h1 {
  font-size: 24px;
  font-weight: normal;
}

.langs {
  max-width: 150px;
  height: 38px;
  float: right;
  font-size: 12px;
  margin-right: 4px;
}

footer {
  min-height: 38px;
  line-height: 38px;
  font-size: 12px;
  color: #999;
  text-align: center;
  margin: 0 0 10px 10px;
}
</style>
