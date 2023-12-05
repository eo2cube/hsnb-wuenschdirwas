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
    <ol-projection-register
      projectionName="EPSG:32633"
      projectionDef="+proj=utm +zone=33 +datum=WGS84 +units=m +no_defs +type=crs"
    />
    
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

    <ol-webgl-tile-layer :style="evi" ref="layer">
      <ol-source-geo-tiff ref="source"
        :sources="[
          {
            // alternative day: 2018/6/S2B_33UUV_20180626_0_L2A/B04.tif
            url: 'https://sentinel-cogs.s3.us-west-2.amazonaws.com/sentinel-s2-l2a-cogs/33/U/UV/2023/7/S2A_33UUV_20230715_0_L2A/B04.tif',
            max: 10000,
          },
          {
            url: 'https://sentinel-cogs.s3.us-west-2.amazonaws.com/sentinel-s2-l2a-cogs/33/U/UV/2023/7/S2A_33UUV_20230715_0_L2A/B03.tif',
            max: 10000,
          },
          {
            url: 'https://sentinel-cogs.s3.us-west-2.amazonaws.com/sentinel-s2-l2a-cogs/33/U/UV/2023/7/S2A_33UUV_20230715_0_L2A/B02.tif',
            max: 10000,
          },
          {
            url: 'https://sentinel-cogs.s3.us-west-2.amazonaws.com/sentinel-s2-l2a-cogs/33/U/UV/2023/7/S2A_33UUV_20230715_0_L2A/B08.tif',
            max: 10000,
          },
        ]"
      />
    </ol-webgl-tile-layer>
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

const center = ref([383825.626, 5966042.979]);  // field known by Sina
const projection = ref("EPSG:32633");
const zoom = ref(14);
const rotation = ref(0);

const max = 3000;
function normalize(value) {
  return ["/", value, max];
}
const red = normalize(["band", 1]);
const green = normalize(["band", 2]);
const blue = normalize(["band", 3]);
const trueColor = ref({
  color: ["array", red, green, blue, 1],
  gamma: 1.1,
});

// band numbers according to the order in the `sources` array above (1-based)
const NDVI_FORMULA = ['/', ['-', ['band', 4], ['band', 1]], ['+', ['band', 4], ['band', 1]]];

// constants for EVI calculation
const L  = 1;
const C1 = 6;
const C2 = 7.5;
const G  = 2.5;

// formula: G * (nir-red) / (nir + C1*red - C2*blue + L)
const EVI_FORMULA = [
  '*', G, ['/', 
                ['-', ['band', 4], ['band', 1]],
                ['+',
                      ['-',
                            ['+', ['band', 4], ['*', C1, ['band', 1]]],
                                               ['*', C2, ['band', 3]]
                      ],
                L]
          ]
];

// color ramp for vegetation index
const colorstops = [
  -0.2,  [191, 191, 191],
  -0.1,  [219, 219, 219],
  0,     [255, 255, 224],
  0.025, [255, 250, 204],
  0.05,  [237, 232, 181],
  0.075, [222, 217, 156],
  0.1,   [204, 199, 130],
  0.125, [189, 184, 107],
  0.15,  [176, 194, 97],
  0.175, [163, 204, 89],
  0.2,   [145, 191, 82],
  0.25,  [128, 179, 71],
  0.3,   [112, 163, 64],
  0.35,  [97, 150, 54],
  0.4,   [79, 138, 46],
  0.45,  [64, 125, 36],
  0.5,   [48, 110, 28],
  0.55,  [33, 97, 18],
  0.6,   [15, 84, 10],
  0.65,  [0, 69, 0]
];

const ndvi = {
  color: [
    'interpolate',
    ['linear'],
    NDVI_FORMULA,
    ...colorstops
  ],
};

const evi = {
  color: [
    'interpolate',
    ['linear'],
    EVI_FORMULA,
    ...colorstops
  ],
};

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
  let baseurl = 'https://phenocube.org/api/stac';
  let query = '{"eo%3Acloud_cover"%3A{"lt"%3A"' + cloudcover.value + '"}}';
  let url = baseurl + '/search' + '?query=' + query + '&bbox=' + bbox.value;
  let response = await fetch(url);
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
