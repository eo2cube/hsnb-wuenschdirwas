<template>
  <header>
    <h1>AgriSens DEMMIN 4.0</h1>
  </header>

  <h2>Filters</h2>

  <label>Maximum cloud coverage:</label>
  <input v-model="cloudcover" @input="handleFilterChange" type="range" min="0" max="100" />
  <input v-model="cloudcover" @input="handleFilterChange"/>%<br>

  <label>Bounding box:</label>
  {{ bbox }}

  <ol-map style="height: 400px/*100cqh*/" ref="map" @moveend="handleFilterChange">
    <ol-view
      ref="view"
      :center="center"
      :rotation="rotation"
      :zoom="zoom"
      :projection="projection"
    />
    <ol-tile-layer>
      <ol-source-osm />
    </ol-tile-layer>
  </ol-map>

  <h2>Results ({{ results.context.returned }} out of {{ results.context.matched }} matching)</h2>

  <ul id="resultlist">
    <li v-for="f in results.features">
      <img :src="f.assets.thumbnail.href" class="thumbnail"/>
      <h3>{{ f.properties.datetime.substr(0,10) }}</h3>
      <a :href="f.assets.thumbnail.href">Thumbnail</a>
      <a :href="f.assets.visual.href">TCI COG</a>
    </li>
  </ul>

  <footer>Legal stuff</footer>
</template>

<script setup>
import { ref } from "vue";
import {fromLonLat, toLonLat} from "ol/proj";

const center = ref(fromLonLat([13.0328, 53.9071]));  // center of Demmin (in OSM)
const projection = ref("EPSG:3857");
const zoom = ref(10);
const rotation = ref(0);

const cloudcover = ref(30);
const bbox = ref('');

const results = ref({  // structure as returned from STAC endpoint
  context: {
    limit: 10,
    matched: 0,
    returned: 0
  },
  features: [],
});

const view = ref(null);
const map = ref(null);

async function handleFilterChange() {
  let ext = view.value.calculateExtent(map.value.map.getSize());
  bbox.value = toLonLat([ext[0], ext[1]]) + ',' + toLonLat([ext[2], ext[3]]);
  let response = await fetch('https://phenocube.org/api/stac/search?query={"eo%3Acloud_cover"%3A{"lt"%3A"' + cloudcover.value + '"}}&bbox=' + bbox.value);
  results.value = await response.json();
}
</script>

<style scoped>
header {
  line-height: 1.5;
}

@media (min-width: 1024px) {
  header {
    display: flex;
    place-items: center;
    padding-right: calc(var(--section-gap) / 2);
  }
}

ul#resultlist {
  overflow-x: scroll;
  padding: 0;
}

#resultlist li {
  float: left;
  list-style: none;
  margin-left: 20px;
}

#resultlist img.thumbnail {
  height: 100px;
  border: 1px solid black;
}

#resultlist img,
#resultlist h3,
#resultlist a {
  display: block;
}
</style>
