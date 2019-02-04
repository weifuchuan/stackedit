<template>
  <modal-inner aria-label="Insert image">
    <div class="modal__content">
      <p>
        Please provide a
        <b>URL</b> for your image.
      </p>
      <form-entry label="URL" error="url">
        <input
          slot="field"
          class="textfield"
          type="text"
          v-model.trim="url"
          @keydown.enter="resolve()"
        >
      </form-entry>
      <input
        ref="uploadInput"
        class="upload-input"
        type="file"
        accept="image/*"
        @change="onSelectFile"
      >
      <menu-entry @click.native="clickUpload">
        <div>Upload Image</div>
      </menu-entry>
      <menu-entry
        @click.native="openGooglePhotos(token)"
        v-for="token in googlePhotosTokens"
        :key="token.sub"
      >
        <icon-provider slot="icon" provider-id="googlePhotos"></icon-provider>
        <div>Open from Google Photos</div>
        <span>{{token.name}}</span>
      </menu-entry>
      <menu-entry @click.native="addGooglePhotosAccount">
        <icon-provider slot="icon" provider-id="googlePhotos"></icon-provider>
        <span>Add Google Photos account</span>
      </menu-entry>
    </div>
    <div class="modal__button-bar">
      <button class="button" @click="reject()">Cancel</button>
      <button class="button button--resolve" @click="resolve()">Ok</button>
    </div>
  </modal-inner>
</template>

<script>
import modalTemplate from "./common/modalTemplate";
import MenuEntry from "../menus/common/MenuEntry";
import googleHelper from "../../services/providers/helpers/googleHelper";
import store from "../../store";

import axios from "axios";
import EventEmitter from "wolfy87-eventemitter";
import { fromEvent } from "rxjs";
import qs from "qs";

const url = window.location.href;
let params = url.substring(url.lastIndexOf("?") + 1);
params = qs.parse(params);
let { uploadAction, DEV, staticBaseUrl } = params;

// uploadAction = "http://localhost:8080/file/upload";
// DEV = true;
// staticBaseUrl = "http://localhost:8080";

export default modalTemplate({
  components: {
    MenuEntry
  },
  data: () => ({
    url: ""
  }),
  computed: {
    googlePhotosTokens() {
      const googleTokensBySub = store.getters["data/googleTokensBySub"];
      return Object.values(googleTokensBySub)
        .filter(token => token.isPhotos)
        .sort((token1, token2) => token1.name.localeCompare(token2.name));
    }
  },
  methods: {
    resolve() {
      if (!this.url) {
        this.setError("url");
      } else {
        const { callback } = this.config;
        this.config.resolve();
        callback(this.url);
      }
    },
    reject() {
      const { callback } = this.config;
      this.config.reject();
      callback(null);
    },
    addGooglePhotosAccount() {
      return googleHelper.addPhotosAccount();
    },
    async openGooglePhotos(token) {
      const { callback } = this.config;
      this.config.reject();
      const res = await googleHelper.openPicker(token, "img");
      if (res[0]) {
        store.dispatch("modal/open", {
          type: "googlePhoto",
          url: res[0].url,
          callback
        });
      }
    },
    async onSelectFile(e) {
      const fileList = this.$refs.uploadInput.files;
      if (fileList.length > 0) {
        const file = fileList.item(0);
        const { response } = upload([file]);        
        const ret = (await response).data;
        let uri = ret[file.name];
        if (DEV) {
          uri = staticBaseUrl + uri;
        }
        this.url = uri;
      }
    },
    clickUpload() {
      this.$refs.uploadInput.click();
    }
  }
});

function upload(files, otherParams = {}) {
  const form = new FormData();
  for (let file of files) {
    form.append("file", file);
  }
  for (let key in otherParams) {
    form.append(key, otherParams[key]);
  }
  const uploadProgress = new EventEmitter();
  const progress = fromEvent(uploadProgress, "uploadProgress");
  const response = axios.post(uploadAction, form, {
    headers: {
      "Content-Type": "multipart/form-data"
    },
    onUploadProgress: evt => {
      let completed = Math.round((evt.loaded * 100) / evt.total);
      uploadProgress.emit("uploadProgress", completed);
    }
  });
  return {
    response,
    progress
  };
}
</script>

<style lang="scss" scoped>
.upload-input {
  display: none;
}
</style>
