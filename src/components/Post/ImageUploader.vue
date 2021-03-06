<template>
  <div class="content">
    <silentbox-group v-if="clips.length">
      <template v-for="(clip, index) in clips">
        <div class="clip" :key="index">
          <silentbox-item :src="clip.urlLarge">
            <template v-show="!clip.canceled">
              <img :src="clip.urlThumbnail" :class="{ inProgress: clip.progress < 100 }">
              <progress v-if="clip.progress < 100" class="progress is-info" :value="clip.progress" max="100" />
            </template>
          </silentbox-item>

          <a v-if="!post.clips_attributes[index]._destroy" class="button is-small" @click="removeClip(index)">
            <font-awesome-icon icon="trash" />
          </a>
          <a v-else class="button is-small" @click="restoreClip(index)">
            <font-awesome-icon icon="circle" />
          </a>
        </div>
      </template>
    </silentbox-group>

    <div class="file">
      <label class="file-label">
        <input class="file-input" type="file" @change="onFileChange" multiple>
        <span class="file-cta">
          <span class="file-icon">
            <i class="fas fa-upload"></i>
          </span>
          <span class="file-label">
            Upload images
          </span>
        </span>
      </label>
    </div>
  </div>
</template>

<script>
import axios from 'axios'
import jsonToFormData from '@/utils/object-to-formdata'
import ThumbnailGenerator from '@/utils/ThumbnailGenerator'

export default {
  name: 'ImageUploader',

  props: ['post', 'clips'],

  data: () => {
    return {
    }
  },

  methods: {
    onFileChange (e) {
      var files = e.target.files || e.dataTransfer.files
      if (!files.length) { return }

      for (var file of files) {
        this.uploadImage(file)
      }
    },

    uploadImage (file) {
      var vm = this

      var clipIndex = vm.post.clips_attributes.length
      vm.post.clips_attributes.push({
        image: null
      })

      const blankImage = 'data:image/gif;base64,R0lGODlhAQABAIAAAP///wAAACwAAAAAAQABAAACAkQBADs='
      vm.clips.push({
        progress: 0,
        canceled: false,
        cancelSource: axios.CancelToken.source(),
        urlLarge: blankImage,
        urlThumbnail: blankImage
      })

      var reader = new FileReader()
      reader.onload = (e) => {
        var generator = new ThumbnailGenerator()
        generator.createThumbnail(e.target.result, 256).then(function (base64Thumbnail) {
          vm.clips[clipIndex].urlThumbnail = base64Thumbnail
        })
        generator.createThumbnail(e.target.result, 1200).then(function (base64Thumbnail) {
          vm.clips[clipIndex].urlLarge = base64Thumbnail
        })

        this.presign(file)
          .then((data) => {
            this.s3Upload(data.url, data.fields, file, clipIndex)
              .then(() => {
                var uploadedFileData = JSON.stringify({
                  id: data.fields['key'].match(/cache\/(.+)/)[1], // remove the Shrine storage prefix
                  storage: 'cache',
                  metadata: {
                    size: file.size,
                    filename: file.name,
                    mime_type: file.type
                  }
                })

                vm.post.clips_attributes[clipIndex].image = uploadedFileData
              })
              .catch((error) => {
                if (axios.isCancel(error)) {
                  console.log('Upload canceled', file.name)
                } else {
                  console.log('Upload failed', error)
                }
              })
          })
          .catch((error) => {
            console.log('Presign failed', error)
          })
      }

      reader.readAsDataURL(file)
    },

    presign (file) {
      return this.$http.get('/presign?filename=' + file.name)
        .then(response => {
          return response.data
        })
    },

    s3Upload (url, fields, file, clipIndex) {
      var vm = this

      var formData = jsonToFormData(fields)
      formData.append('file', file)

      const axiosS3 = axios.create({
        baseURL: url
      })

      const axiosOptions = {
        method: 'POST',
        data: formData,
        cancelToken: vm.clips[clipIndex].cancelSource.token,
        onUploadProgress: (progressEvent) => {
          var percentCompleted = Math.round(progressEvent.loaded * 100 / progressEvent.total)
          vm.clips[clipIndex].progress = percentCompleted
        }
      }

      return axiosS3(axiosOptions)
    },

    removeClip (index) {
      const clip = this.post.clips_attributes[index]
      if (clip.id == null) {
        this.clips[index].canceled = true
        this.clips[index].cancelSource.cancel()
      }
      this.post.clips_attributes[index]._destroy = true
    },

    restoreClip (index) {
      this.post.clips_attributes[index]._destroy = 0
    }
  }
}
</script>

<style lang="sass">
.clip
  position: relative

  img
    &.inProgress
      opacity: 0.3

  progress
    position: absolute
    top: 0
    bottom: 0
    height: 5px
    left: 0
    right: 0
    width: 90%
    margin: auto

  .button
    position: absolute
    top: 10px
    left: 10px
    display: none

  &:hover
    .button
      display: block
</style>
