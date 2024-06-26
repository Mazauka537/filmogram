<template>
  <template v-if="collection">

    <ScrollableBlock ref="scrollableBlock">

      <div class="view-collection">

        <HeadBar :title="collection.title" :scrollable-block="scrollableBlock" :scroll-height="130"/>

        <div class="view-collection__header">

          <div class="view-collection__preview">
            <CollectionPreview :scrollable-block="scrollableBlock" :sorted-films="sortedFilms" :image-path="imagePath"/>
          </div>

          <div class="view-collection__info">
            <div class="view-collection__open">
              {{ collection.public ? 'Открытая коллеция' : 'Закрытая коллекция' }}
            </div>
            <div class="view-collection__name">{{ collection.title }}</div>
            <div class="view-collection__desc" :class="{'view-collection__desc_wrapped': isDescWrapped}" @click="isDescWrapped = !isDescWrapped">{{ collection.description }}</div>
            <div class="view-collection__main-info">
              <router-link :to="'/user/' + collection.user_id" class="view-collection__owner">
                <span class="view-collection__owner-avatar">
                  <img
                      :src="collection.user.avatar ? env.VUE_APP_AVATARS_PATH + collection.user.avatar : '/img/user.jpg'"
                      alt="avatar">
                </span>
                <span class="view-collection__owner-name">{{ collection.user.name }}</span>
              </router-link>&nbsp;&bull;
              <span class="view-collection__likes">{{ collection.saves_count }} {{ likesDeclination }}</span>&nbsp;&bull;
              <span class="view-collection__films_count">{{ collection.films_count }} {{ filmsDeclination }}</span>
            </div>
          </div>
        </div>

        <div class="view-collection__body">
          <div class="view-collection__ctrl-panel" v-if="!collection.constant">
            <span class="view-collection__btn" v-if="$store.getters['auth/isOwner'](collection.user_id)">
            <MyButton @click="$router.push({path: '/collection/' + $route.params.id, query: {popUp: 'addFilms'}})">
              Дополнить
            </MyButton>
            </span>
            <button class="view-collection__save-btn"
                    v-if="!$store.getters['auth/isOwner'](collection.user.id)"
                    @click="toggleSave(collection)">
              <SaveBtn :active="collection.isInSaves"/>
            </button>
            <button class="view-collection__share-btn">
              <ShareBtn :link="window.location.origin + '/collection/' + collection.id"/>
            </button>
            <button class="view-collection__more-btn" v-if="$store.getters['auth/isOwner'](collection.user.id)">
              <MoreBtn :options="moreBtnOptions"/>
            </button>
          </div>

          <div class="view-collection__films-table-head">
            <FilmTableHead/>
          </div>

          <LoadableItemsContainer :loader="collectedFilmsLoader" style="margin-top: 15px;"
                                  :scrollable-block="scrollableBlock">
            <DragContainer @drop="changeOrder"
                           :disable="!$store.getters['auth/isOwner'](collection.user_id)"
                           :scrollable-block="scrollableBlock">
              <DragBlock v-for="(film, index) in sortedFilms" :key="film.id" :is-grab-visible="isFilmsOrderChangeable">
                <div class="view-collection__film-block-wrapper" ref="elemsFilmBlocks" :data-film-id="film.id">
                  <FilmBlock :film="film"
                             style="margin: 5px 0"
                             :number="index + 1"
                             :collection="collection"
                             :is-more-btn-hidden="isFilmsOrderChangeable"
                             @save="toggleFavorite"
                             @click="$router.push({query: {film: film.filmKp.kinopoiskId}})"
                             @delete="deleteFilm(film)"
                             @addToCollection="addingToCollectionFilm = film; $router.push({query: {popUp: 'addFilmToCollections'}})"/>
                </div>
              </DragBlock>
            </DragContainer>
          </LoadableItemsContainer>
        </div>
      </div>

    </ScrollableBlock>


    <SideFilmBlock/>


    <PopUpsContainer>
      <PopUpEditCollection v-if="$route.query.popUp === 'editCollection'"
                           :collection="collection"
                           @collectionEdited="onCollectionEdited"/>
      <PopUpDeleteCollection v-else-if="$route.query.popUp === 'deleteCollection'"
                             :collection="collection"/>
      <PopUpAddFilms v-else-if="$route.query.popUp === 'addFilms'"
                     :collection="collection"
                     @close="collectedFilmsLoader.reset()"/>
      <PopUpAddFilmToCollections v-else-if="$route.query.popUp === 'addFilmToCollections'"
                                 :film="addingToCollectionFilm"
                                 @favoriteCollectionChanged="toggleFavorite"
                                 @close="onCurrentCollectionChanged"/>
    </PopUpsContainer>

  </template>
</template>

<script>
import MyButton from "@/components/UI/MyButton";
import DragBlock from "@/components/DragBlock";
import DragContainer from "@/components/DragContainer";
import LoadableItemsContainer from "@/components/LoadableItemsContainer";
import {ref, computed, onMounted, watch} from "vue";
import {useRoute} from "vue-router";
import useDeleteFilmFromCollection from "@/composables/useDeleteFilmFromCollection";
import useChangeFilmOrder from "@/composables/useChangeFilmOrder";
import FilmBlock from "@/components/FilmBlock";
import FilmTable from "@/components/FilmTable";
import FilmTableHead from "@/components/FilmTableHead";
import MoreBtn from "@/components/UI/MoreBtn";
import ShareBtn from "@/components/UI/ShareBtn";
import useToggleCollectionPublic from "@/composables/useToggleCollectionPublic";
import useToggleSave from "@/composables/useToggleSave";
import SaveBtn from "@/components/UI/SaveBtn";
import useToggleFavorite from "@/composables/useToggleFavorite";
import useGetCollection from "@/composables/useGetCollection";
import ScrollableBlock from "@/components/ScrollableBlock";
import {useRouter} from "vue-router/dist/vue-router";
import {defineAsyncComponent} from "vue";
import PopUpsContainer from "@/components/popUps/PopUpsContainer";
import HeadBar from "@/components/HeadBar";
import SideFilmBlock from "@/components/SideFilmBlock";
import CollectionPreview from "@/components/CollectionPreview";
import useCollectionDataDeclination from "@/composables/useCollectionDataDeclination";

export default {
  components: {
    CollectionPreview,
    SideFilmBlock,
    HeadBar,
    PopUpsContainer,
    PopUpAddFilmToCollections: defineAsyncComponent(() => import('@/components/popUps/PopUpAddFilmToCollections')),
    PopUpAddFilms: defineAsyncComponent(() => import('@/components/popUps/PopUpAddFilms')),
    PopUpEditCollection: defineAsyncComponent(() => import('@/components/popUps/PopUpEditCollection')),
    PopUpDeleteCollection: defineAsyncComponent(() => import('@/components/popUps/PopUpDeleteCollection')),
    ScrollableBlock,
    SaveBtn,
    ShareBtn,
    MoreBtn,
    FilmTableHead,
    FilmTable,
    FilmBlock, DragContainer, DragBlock, MyButton, LoadableItemsContainer
  },
  setup() {
    const elemsFilmBlocks = ref(undefined)
    const scrollableBlock = ref(undefined)

    const isDescWrapped = ref(true)
    const addingToCollectionFilm = ref(undefined)
    const isFilmsOrderChangeable = ref(false)
    const imageUpdater = ref('?=1')

    const imagePath = computed(() => {
      return collection.value?.image ? process.env.VUE_APP_COLLECTION_IMG_PATH + collection.value.image + imageUpdater.value : undefined
    })

    const sortedFilms = computed(() => {
      return collectedFilmsLoader.items.sort((filmA, filmB) => filmB.order - filmA.order)
    })

    const route = useRoute()
    const router = useRouter()
    const {collection, collectedFilmsLoader, getCollection} = useGetCollection()
    const {deleteFilm} = useDeleteFilmFromCollection(collection, collectedFilmsLoader, elemsFilmBlocks)
    const {changeOrder} = useChangeFilmOrder(sortedFilms)
    const {toggleCollectionPublic} = useToggleCollectionPublic()
    const {toggleSave} = useToggleSave()
    const {toggleFavorite} = useToggleFavorite()
    const {likesDeclination, filmsDeclination} = useCollectionDataDeclination(collection)

    const onCollectionEdited = (newCollection) => {
      collection.value = newCollection
      imageUpdater.value += '1'
      router.back()
    }

    const onCurrentCollectionChanged = (isCurrentCollectionToggled) => {
      if (isCurrentCollectionToggled) {
        collectedFilmsLoader.reset()
      }
    }

    const toggleChangeFilmsOrderMode = () => {
      isFilmsOrderChangeable.value = !isFilmsOrderChangeable.value
    }

    const moreBtnOptions = [{
      text: () => 'Изменить сведения',
      onClick: () => {
        router.push({query: {popUp: 'editCollection'}})
      }
    }, {
      text: () => collection.value.public ? 'Сделать приватной' : 'Сделать общедоступной',
      onClick: () => {
        toggleCollectionPublic(collection.value)
      }
    }, {
      text: () => 'Изменить порядок',
      onClick: () => {
        toggleChangeFilmsOrderMode()
      }
    }, {
      text: () => 'Удалить коллекцию',
      onClick: () => {
        router.push({query: {popUp: 'deleteCollection'}})
      }
    }]

    watch(() => route.params.id, () => {
      getCollection(route.params.id)
    })

    onMounted(() => {
      getCollection(route.params.id)
    })

    return {
      imagePath,
      likesDeclination,
      filmsDeclination,
      onCollectionEdited,
      onCurrentCollectionChanged,
      scrollableBlock,
      moreBtnOptions,
      collection,
      isDescWrapped,
      addingToCollectionFilm,
      collectedFilmsLoader,
      sortedFilms,
      changeOrder,
      deleteFilm,
      toggleSave,
      toggleFavorite,
      elemsFilmBlocks,
      isFilmsOrderChangeable,
      window,
      env: process.env
    }
  }
  }
</script>

<style scoped lang="scss">
@import "src/assets/styles/vars";

.view-collection {
  background: $color-bg-header;
  padding-right: 312px;

  &__header {
    display: flex;
    justify-content: left;
    align-items: end;
    padding: 30px 30px 24px 30px;
  }

  &__preview {
    height: 240px;
    width: 240px;
    flex-shrink: 0;
  }

  &__info {
    padding-left: 25px;
    overflow: hidden;
  }

  &__open {
    text-transform: uppercase;
    color: $color-text-light;
    font-weight: 700;
    font-size: 13px;
    letter-spacing: 0.1px;
  }

  &__name {
    overflow: hidden;
    white-space: nowrap;
    text-overflow: ellipsis;
    font-size: 68px;
    color: $color-text-light;
    font-weight: 700;
    padding-top: 20px;
  }

  &__desc {
    font-size: 13px;
    padding-bottom: 40px;

    &_wrapped {
      overflow: hidden;
      white-space: nowrap;
      text-overflow: ellipsis;
    }
  }

  &__main-info {
    color: $color-text-light;

    a:hover {
      text-decoration: underline;
    }
  }

  &__owner-avatar {
    padding-right: 5px;

    * {
      vertical-align: middle;
    }

    img {
      height: 25px;
      width: 25px;
      border-radius: 50%;
      background: $color-bg-side;
    }
  }

  &__body {
    background: $color-bg-body;
    min-height: 400px;
    padding: 30px 30px 200px 30px;
  }

  &__ctrl-panel {
    padding-bottom: 30px;

    * {
      vertical-align: middle;
    }
  }

  &__btn {
    margin-right: 25px;
  }

  &__save-btn, &__share-btn, &__more-btn {
    display: inline-block;
    height: 40px;
    width: 40px;
    cursor: pointer;

    svg {
      fill: $color-text-light;
    }
  }

  &__save-btn {
    margin-right: 20px;

    &_active {
      svg {
        fill: $color-main;
      }
    }
  }

  &__share-btn {
    margin-right: 20px;
  }

  &__film-block-wrapper {

  }
}

@media screen and (max-width: 1460px) {
  .view-collection {

    &__preview {
      height: 180px;
      width: 180px;
    }

    &__name {
      font-size: 48px;
      padding-top: 15px;
    }

    &__desc {
      padding-bottom: 37px;
    }

    &__ctrl-panel {
      padding-bottom: 20px;
    }

    &__body {
      padding: 20px 30px 200px 30px;
    }
  }
}

@media screen and (max-width: 1280px) {
  .view-collection {
    padding-right: 12px;
  }
}

@media screen and (max-width: 1030px) {
  .view-collection {

    &__header {
      padding: 15px;
    }

    &__ctrl-panel {
      padding-bottom: 15px;
    }

    &__body {
      padding: 15px 15px 200px 15px;
    }
  }
}

@media screen and (max-width: 900px) {
  .view-collection {
    &__ctrl-panel {
      padding-bottom: 0;
    }

    &__films-table-head {
      display: none;
    }
  }
}

@media screen and (max-width: 560px) {
  .view-collection {
    background: $color-bg-header-media2;

    &__header {
      display: block;
      padding: 15px 15px 0 15px;
    }

    &__preview {
      margin: 0 auto;
    }

    &__info {
      padding-left: 0;
    }

    &__open {
      display: none;
    }

    &__name {
      font-size: 16px;
      padding: 10px 0 5px 0;
    }

    &__desc {
      padding-bottom: 10px;
    }

    &__body {
      background: none;
    }

    &__btn {
      margin-right: 15px;
    }

    &__more-btn, &__share-btn {
      height: 36px;
      width: 36px;
    }

    &__share-btn {
      margin-right: 10px;
    }
  }
}
</style>
