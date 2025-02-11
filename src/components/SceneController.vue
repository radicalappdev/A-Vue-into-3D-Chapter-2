<template>
  <canvas
    id="bjsCanvas"
    ref="bjsCanvas"
    style="height: calc(50vw); width: calc(100vw)"
  />
  <div>
    <button @click="exportFavData()">Export Favorites (Text File)</button>

    <button @click="copyFavData()">Get Favorites as Text</button>
  </div>
  <textarea
    rows="24"
    cols="100"
    readonly
    v-on:focus="$event.target.select()"
    ref="clone"
    :value="favText"
    type="textarea"
  />
</template>

<script>
import LibraryService from "@/services/LibraryService.js";
import SceneWrapper from "@/scenes/SceneWrapper.js";

export default {
  name: "BabylonScene",
  data() {
    return {
      itemResponse: null,
      totalItems: 0,
      loading: true,
      errored: false,
      page: 1,
      perPage: 12,
      dataSource: "api",
      favorites: [],
      favText: "Click 'Get Favorites as Text'",
    };
  },
  methods: {
    loadAPIData() {
      this.loading = true;
      this.itemResponse = null;
      return LibraryService.getItems(this.perPage, this.page)
        .then((response) => {
          this.itemResponse = response;
          this.totalItems = response.headers["x-wp-total"];
        })
        .catch((error) => {
          console.log(error);
        })
        .finally(() => {
          this.loading = false;
          this.populateWithAPI();
        });
    },
    loadFavData() {
      this.loading = true;
      const storage = { ...localStorage };
      const keys = Object.keys(storage);
      let loadedFavorites = [];
      for (let key of keys) {
        if (key.toString().substring(0, 5) === "item-") {
          const item = JSON.parse(localStorage[key]);
          if (item.isFavorite) {
            loadedFavorites.push(item);
          }
        }
      }
      const start = this.perPage * this.page - 12;
      const end = this.perPage * this.page;
      this.favorites = loadedFavorites.slice(start, end);
      this.populateWithFAV();
      this.loading = false;
    },
    pageAPIData() {
      if (this.page > 1) {
        this.page = this.page - 1;
        if (this.dataSource === "api") {
          this.loadAPIData();
        } else {
          this.loadFavData();
        }
      }
    },
    pageNext() {
      if (this.page < this.itemResponse.headers["x-wp-totalpages"]) {
        this.page = this.page + 1;
        if (this.dataSource === "api") {
          this.loadAPIData();
        } else {
          this.loadFavData();
        }
      }
    },
    populateWithAPI() {
      const items = this.itemResponse.data?.map((item) => {
        let fav = false;
        if (localStorage[this.storageKey(item.id)]) {
          const stored = JSON.parse(localStorage[this.storageKey(item.id)]);
          fav = stored?.isFavorite == true ? stored.isFavorite : false;
        }

        return {
          id: item.id,
          title: item.title.rendered,
          description: item.acf.acf_library_description,
          link: item.acf.acf_library_url,
          image: item.featured_image_src_large[0],
          isFavorite: fav,
        };
      });
      SceneWrapper.populateCompactCards(items, this.toggleFavorite);
    },
    populateWithFAV() {
      const items = this.favorites;
      SceneWrapper.populateCompactCards(items, this.toggleFavorite);
    },

    useAPI() {
      this.page = 1;
      this.dataSource = "api";
      console.log("loadAPIData");
      this.loadAPIData();
      // this.populateWithAPI();
    },
    useFavorite() {
      this.page = 1;
      this.dataSource = "fav";
      this.loadFavData();
      console.log("loadFavorite");
      // this.populateWithFAV()();
    },
    toggleFavorite(item, isFavorite) {
      let newItem = item;
      newItem["isFavorite"] = isFavorite;
      localStorage[this.storageKey(item.id)] = JSON.stringify(newItem);
      console.log(localStorage[this.storageKey(item.id)]);
    },
    storageKey(id) {
      return `item-${id}`;
    },

    prepareExportData() {
      const storage = { ...localStorage };
      const keys = Object.keys(storage);
      let loadedFavorites = [];
      for (let key of keys) {
        if (key.toString().substring(0, 5) === "item-") {
          const item = JSON.parse(localStorage[key]);
          if (item.isFavorite) {
            loadedFavorites.push(item);
          }
        }
      }
      console.log(loadedFavorites);
      const exportItems = loadedFavorites
        ?.map((item) => {
          return `${item.title}\n     ${item.link}`;
        })
        .join("\n\n");
      return exportItems;
    },

    async copyFavData() {
      this.favText = await this.prepareExportData();
      // this.$refs.clone.focus();
      // document.execCommand("copy");
      // this.clipboard.writeText(this.prepareExportData());
    },

    // A stub to export favorites from LocalStorage to a JSON file.
    exportFavData() {
      const exportItems = this.prepareExportData();

      let exportEl = document.createElement("a");
      exportEl.setAttribute(
        "href",
        "data:text/plain;charset=utf-8," + encodeURIComponent(exportItems)
      );
      exportEl.style.display = "none";
      exportEl.setAttribute("download", "ec-labs-favorites.txt");
      document.body.appendChild(exportEl);
      exportEl.click();
      document.body.removeChild(exportEl);
    },
  },
  computed: {},
  created() {
    this.loadAPIData();
  },
  async mounted() {
    await SceneWrapper.createScene(document.getElementById("bjsCanvas"));
    SceneWrapper.sendControlPanelButton("<", this.pageAPIData);
    SceneWrapper.sendControlPanelButton(">", this.pageNext);
    SceneWrapper.sendControlPanelButton("", () => {});
    SceneWrapper.sendControlPanelButton("API", this.useAPI);
    SceneWrapper.sendControlPanelButton("FAV", this.useFavorite);
  },
};
</script>