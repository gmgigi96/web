<template>
  <div>
    <list-loader v-if="loading" />
    <template v-else>
      <no-content-message v-if="isEmpty" id="files-favorites-empty" class="files-empty" icon="star">
        <template #message>
          <span v-translate>There are no resources marked as favorite</span>
        </template>
      </no-content-message>
      <oc-table-files
        v-else
        id="files-favorites-table"
        v-model="selected"
        class="files-table"
        :class="{ 'files-table-squashed': isSidebarOpen }"
        :are-paths-displayed="true"
        :are-thumbnails-displayed="displayThumbnails"
        :resources="activeFiles"
        :target-route="targetRoute"
        :highlighted="highlightedFile ? highlightedFile.id : null"
        :header-position="headerPosition"
        @showDetails="$_mountSideBar_showDetails"
        @fileClick="$_fileActions_triggerDefaultAction"
        @rowMounted="rowMounted"
      >
        <template #quickActions="props">
          <quick-actions class="oc-visible@s" :item="props.resource" :actions="app.quickActions" />
        </template>
        <template #footer>
          <pagination />
          <list-info
            v-if="activeFiles.length > 0"
            class="uk-width-1-1 oc-my-s"
            :files="totalFilesCount.files"
            :folders="totalFilesCount.folders"
            :size="totalFilesSize"
          />
        </template>
      </oc-table-files>
    </template>
  </div>
</template>

<script>
import { mapGetters, mapState, mapActions, mapMutations } from 'vuex'

import { buildResource } from '../helpers/resources'
import FileActions from '../mixins/fileActions'
import MixinFilesListPositioning from '../mixins/filesListPositioning'
import MixinFilesListPagination from '../mixins/filesListPagination'
import MixinMountSideBar from '../mixins/sidebar/mountSideBar'
import { VisibilityObserver } from 'web-pkg/src/observer'
import { ImageDimension, ImageType } from '../constants'
import debounce from 'lodash-es/debounce'

import QuickActions from '../components/FilesList/QuickActions.vue'
import ListLoader from '../components/FilesList/ListLoader.vue'
import NoContentMessage from '../components/FilesList/NoContentMessage.vue'
import ListInfo from '../components/FilesList/ListInfo.vue'
import Pagination from '../components/FilesList/Pagination.vue'

const visibilityObserver = new VisibilityObserver()

export default {
  components: { QuickActions, ListLoader, NoContentMessage, ListInfo, Pagination },

  mixins: [FileActions, MixinFilesListPositioning, MixinFilesListPagination, MixinMountSideBar],

  data: () => ({
    loading: true
  }),

  computed: {
    ...mapState(['app']),
    ...mapState('Files', ['files']),
    ...mapGetters('Files', [
      'davProperties',
      'highlightedFile',
      'activeFiles',
      'selectedFiles',
      'inProgress',
      'totalFilesCount',
      'totalFilesSize'
    ]),
    ...mapGetters(['user', 'configuration']),

    isSidebarOpen() {
      return this.highlightedFile !== null
    },

    targetRoute() {
      return { name: 'files-personal' }
    },

    selected: {
      get() {
        return this.selectedFiles
      },
      set(resources) {
        this.SELECT_RESOURCES(resources)
      }
    },

    isEmpty() {
      return this.activeFiles.length < 1
    },

    uploadProgressVisible() {
      return this.inProgress.length > 0
    },

    displayThumbnails() {
      return !this.configuration.options.disablePreviews
    }
  },

  watch: {
    uploadProgressVisible() {
      this.adjustTableHeaderPosition()
    },

    $route: {
      handler: '$_filesListPagination_updateCurrentPage',
      immediate: true
    }
  },

  created() {
    this.loadResources()
    window.onresize = this.adjustTableHeaderPosition
  },

  mounted() {
    this.adjustTableHeaderPosition()
  },

  beforeDestroy() {
    visibilityObserver.disconnect()
  },

  methods: {
    ...mapActions('Files', ['loadIndicators', 'loadPreview']),
    ...mapMutations('Files', ['SELECT_RESOURCES', 'LOAD_FILES', 'CLEAR_CURRENT_FILES_LIST']),

    rowMounted(resource, component) {
      if (!this.displayThumbnails) {
        return
      }

      const debounced = debounce(({ unobserve }) => {
        unobserve()
        this.loadPreview({
          resource,
          isPublic: false,
          dimensions: ImageDimension.Thumbnail,
          type: ImageType.Thumbnail
        })
      }, 250)

      visibilityObserver.observe(component.$el, { onEnter: debounced, onExit: debounced.cancel })
    },

    async loadResources() {
      this.loading = true
      this.CLEAR_CURRENT_FILES_LIST()

      let resources = await this.$client.files.getFavoriteFiles(this.davProperties)

      resources = resources.map(buildResource)
      this.LOAD_FILES({ currentFolder: null, files: resources })
      this.loadIndicators({ client: this.$client, currentFolder: '/' })

      this.loading = false
    }
  }
}
</script>
