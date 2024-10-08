<template>

  <DefaultField :field="field" :errors="errors" :show-help-text="showHelpText" :full-width-content="fullWidthContent">
    <template #field>
      <input :id="field.attribute" type="text" class="w-full form-control form-input form-control-bordered"
        :class="errorClasses" :placeholder="field.name" v-model="value" v-show="this.showMapCoordinates" />
      <div id="map" :style="'height:' + this.height"></div>
    </template>
  </DefaultField>

</template>

<script>

import { FormField, HandlesValidationErrors } from 'laravel-nova'
import "leaflet/dist/leaflet.css";
import * as L from "leaflet";
import * as LS from 'leaflet-geosearch';
import "leaflet-geosearch/assets/css/leaflet.css";
import "leaflet.gridlayer.googlemutant";
import $Scriptjs from "scriptjs";
import "leaflet.markercluster";
import "leaflet.markercluster/dist/MarkerCluster.css";
import "leaflet.markercluster/dist/MarkerCluster.Default.css";

export default {
  mixins: [FormField, HandlesValidationErrors],

  props: ['resourceName', 'resourceId', 'field'],

  methods: {
    /*
     * Set the initial, internal value for the field.
     */
    setInitialValue() {
      this.value = this.field.value || ''
    },

    setValue(latitude, longitude) {

      this.value = (latitude || 0) + ',' + (longitude || 0)
    },

    /*
     * Update the field's internal value.
     */
    handleChange(value) {
      this.setValue(value.latitude, value.longitude)
    },
    /**
     * Fill the given FormData object with the field's internal value.
     */
    fill(formData) {

      formData.append(this.fieldAttribute, this.value || '')
    },

  },
  data() {

    var height
    var showMapCoordinates = this.field.showMapCoordinates ?? true
    let tileProviders = {}
    let providerOptions = {}
    if (this.field.providerOptions) providerOptions = this.field.providerOptions
    let defaultLocation = [-33.950195282757, 18.429565429687504];
    let searchProvider = new LS.EsriProvider({
          params: providerOptions
    });
    var onUpdateCoordinates = defaultLocation
    if (this.field.value) onUpdateCoordinates = this.field.value.split(',')

    // Default location
    if (this.field.defaultLatitude && this.field.defaultLongitude) {
      defaultLocation = [this.field.defaultLatitude, this.field.defaultLongitude];
    }

    // Search Providers
    switch (this.field.searchProvider) {
      case "bing":
        searchProvider = new LS.BingProvider({
          params: {
            key: this.field.searchProviderKey,
          },
        });
        break;
      case "google":
        searchProvider = new LS.GoogleProvider({
          params: {
            key: this.field.searchProviderKey,
          },
        });
        break;
      case "openstreetmap":
        searchProvider = new LS.OpenStreetMapProvider();
        break;
    }

    // Marker Icon
    let base64img =
      "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABkAAAApCAYAAADAk4LOAAAFgUlEQVR4Aa1XA5BjWRTN2oW17d3YaZtr2962HUzbDNpjszW24mRt28p47v7zq/bXZtrp/lWnXr337j3nPCe85NcypgSFdugCpW5YoDAMRaIMqRi6aKq5E3YqDQO3qAwjVWrD8Ncq/RBpykd8oZUb/kaJutow8r1aP9II0WmLKLIsJyv1w/kqw9Ch2MYdB++12Onxee/QMwvf4/Dk/Lfp/i4nxTXtOoQ4pW5Aj7wpici1A9erdAN2OH64x8OSP9j3Ft3b7aWkTg/Fm91siTra0f9on5sQr9INejH6CUUUpavjFNq1B+Oadhxmnfa8RfEmN8VNAsQhPqF55xHkMzz3jSmChWU6f7/XZKNH+9+hBLOHYozuKQPxyMPUKkrX/K0uWnfFaJGS1QPRtZsOPtr3NsW0uyh6NNCOkU3Yz+bXbT3I8G3xE5EXLXtCXbbqwCO9zPQYPRTZ5vIDXD7U+w7rFDEoUUf7ibHIR4y6bLVPXrz8JVZEql13trxwue/uDivd3fkWRbS6/IA2bID4uk0UpF1N8qLlbBlXs4Ee7HLTfV1j54APvODnSfOWBqtKVvjgLKzF5YdEk5ewRkGlK0i33Eofffc7HT56jD7/6U+qH3Cx7SBLNntH5YIPvODnyfIXZYRVDPqgHtLs5ABHD3YzLuespb7t79FY34DjMwrVrcTuwlT55YMPvOBnRrJ4VXTdNnYug5ucHLBjEpt30701A3Ts+HEa73u6dT3FNWwflY86eMHPk+Yu+i6pzUpRrW7SNDg5JHR4KapmM5Wv2E8Tfcb1HoqqHMHU+uWDD7zg54mz5/2BSnizi9T1Dg4QQXLToGNCkb6tb1NU+QAlGr1++eADrzhn/u8Q2YZhQVlZ5+CAOtqfbhmaUCS1ezNFVm2imDbPmPng5wmz+gwh+oHDce0eUtQ6OGDIyR0uUhUsoO3vfDmmgOezH0mZN59x7MBi++WDL1g/eEiU3avlidO671bkLfwbw5XV2P8Pzo0ydy4t2/0eu33xYSOMOD8hTf4CrBtGMSoXfPLchX+J0ruSePw3LZeK0juPJbYzrhkH0io7B3k164hiGvawhOKMLkrQLyVpZg8rHFW7E2uHOL888IBPlNZ1FPzstSJM694fWr6RwpvcJK60+0HCILTBzZLFNdtAzJaohze60T8qBzyh5ZuOg5e7uwQppofEmf2++DYvmySqGBuKaicF1blQjhuHdvCIMvp8whTTfZzI7RldpwtSzL+F1+wkdZ2TBOW2gIF88PBTzD/gpeREAMEbxnJcaJHNHrpzji0gQCS6hdkEeYt9DF/2qPcEC8RM28Hwmr3sdNyht00byAut2k3gufWNtgtOEOFGUwcXWNDbdNbpgBGxEvKkOQsxivJx33iow0Vw5S6SVTrpVq11ysA2Rp7gTfPfktc6zhtXBBC+adRLshf6sG2RfHPZ5EAc4sVZ83yCN00Fk/4kggu40ZTvIEm5g24qtU4KjBrx/BTTH8ifVASAG7gKrnWxJDcU7x8X6Ecczhm3o6YicvsLXWfh3Ch1W0k8x0nXF+0fFxgt4phz8QvypiwCCFKMqXCnqXExjq10beH+UUA7+nG6mdG/Pu0f3LgFcGrl2s0kNNjpmoJ9o4B29CMO8dMT4Q5ox8uitF6fqsrJOr8qnwNbRzv6hSnG5wP+64C7h9lp30hKNtKdWjtdkbuPA19nJ7Tz3zR/ibgARbhb4AlhavcBebmTHcFl2fvYEnW0ox9xMxKBS8btJ+KiEbq9zA4RthQXDhPa0T9TEe69gWupwc6uBUphquXgf+/FrIjweHQS4/pduMe5ERUMHUd9xv8ZR98CxkS4F2n3EUrUZ10EYNw7BWm9x1GiPssi3GgiGRDKWRYZfXlON+dfNbM+GgIwYdwAAAAASUVORK5CYII=";
    delete L.Icon.Default.prototype._getIconUrl;

    let base64ShadowUrl =
      "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAACkAAAApCAQAAAACach9AAACMUlEQVR4Ae3ShY7jQBAE0Aoz/f9/HTMzhg1zrdKUrJbdx+Kd2nD8VNudfsL/Th///dyQN2TH6f3y/BGpC379rV+S+qqetBOxImNQXL8JCAr2V4iMQXHGNJxeCfZXhSRBcQMfvkOWUdtfzlLgAENmZDcmo2TVmt8OSM2eXxBp3DjHSMFutqS7SbmemzBiR+xpKCNUIRkdkkYxhAkyGoBvyQFEJEefwSmmvBfJuJ6aKqKWnAkvGZOaZXTUgFqYULWNSHUckZuR1HIIimUExutRxwzOLROIG4vKmCKQt364mIlhSyzAf1m9lHZHJZrlAOMMztRRiKimp/rpdJDc9Awry5xTZCte7FHtuS8wJgeYGrex28xNTd086Dik7vUMscQOa8y4DoGtCCSkAKlNwpgNtphjrC6MIHUkR6YWxxs6Sc5xqn222mmCRFzIt8lEdKx+ikCtg91qS2WpwVfBelJCiQJwvzixfI9cxZQWgiSJelKnwBElKYtDOb2MFbhmUigbReQBV0Cg4+qMXSxXSyGUn4UbF8l+7qdSGnTC0XLCmahIgUHLhLOhpVCtw4CzYXvLQWQbJNmxoCsOKAxSgBJno75avolkRw8iIAFcsdc02e9iyCd8tHwmeSSoKTowIgvscSGZUOA7PuCN5b2BX9mQM7S0wYhMNU74zgsPBj3HU7wguAfnxxjFQGBE6pwN+GjME9zHY7zGp8wVxMShYX9NXvEWD3HbwJf4giO4CFIQxXScH1/TM+04kkBiAAAAAElFTkSuQmCC";

    if (this.field.markerIcon == null) {
      L.Icon.Default.imagePath = ".";
      L.Icon.Default.mergeOptions({
        iconRetinaUrl: base64img,
        iconUrl: base64img,
        iconSize: [25, 41],
        iconAnchor: [12, 40],
        shadowUrl: base64ShadowUrl,
      });
    } else {
      L.Icon.Default.mergeOptions({
        iconRetinaUrl: this.field.markerIcon,
        iconUrl: this.field.markerIcon,
        iconSize: this.field.markerIconSize,
        iconAnchor: this.field.markerIconAnchor,
        shadowUrl: null,
        shadowAnchor: null,
      });
    }

    // Map height
    if (!this.field.height) {
      height = "300px";
    } else {
      height = this.field.height;
    }

    return {
      tileProviders,
      height,
      defaultLocation,
      searchProvider,
      onUpdateCoordinates,
      showMapCoordinates
    };
  },

  mounted() {
  
    let marker = null;
    var self = this;
    // Tile Layers
    var googleMaps = L.gridLayer.googleMutant()
    var osm
    var defaultTileProvider = this.field.defaultTileProvider || 'openstreetmap'
    var zoom = this.field.zoom || 8

    // Map
    var map = L.map('map', {
    });

    // OpenStreetMap
    osm = L.tileLayer(
      "https://tile.openstreetmap.org/{z}/{x}/{y}.png", {
      attribution: '&copy; <a target="_blank" href="http://osm.org/copyright">OpenStreetMap</a> contributors',
    });
    this.tileProviders["OpenStreetMap"] = osm

    // Google
    if (this.field.googleApiKey && this.field.googleMapType) {
      googleMaps = L.gridLayer.googleMutant({
        maxZoom: 24,
        type: this.field.googleMapType, // valid values are 'roadmap', 'satellite', 'terrain' and 'hybrid'
      });
      this.tileProviders["Google"] = googleMaps
      $Scriptjs(
        "https://maps.googleapis.com/maps/api/js?key=" +
        this.field.googleApiKey +
        "&loading=async"
      );

    }

    // Map Search Provider
    const searchControl = new LS.GeoSearchControl({
      showMarker: false,
      provider: this.searchProvider,
      style: 'bar',
      searchLabel: 'Search for a location',
    });
    map.addControl(searchControl);

    // Default Tile Provider
    switch (defaultTileProvider) {
      case "google":
        if (this.field.googleApiKey && this.field.googleMapType) {
          var baseMaps = {
            "Google": googleMaps,
          }
          map.removeLayer(osm);
          googleMaps.addTo(map);
        }
        break;
      case "openstreetmap":
        var baseMaps = {
          "OpenStreetMap": osm,
        };
        map.removeLayer(googleMaps);
        osm.addTo(map);
        break;
    }

    L.control.layers(baseMaps, null, {
      position: "bottomright",
    }).addTo(map);

    marker = L.marker(L.latLng(this.onUpdateCoordinates))
      .addTo(map)
      .on('click', function (e) {
        marker.bindPopup('Latitude: ' + e.latlng.lat + '<br> Longitude: ' + e.latlng.lng).openPopup();
      });
    map.setView( L.latLng(this.onUpdateCoordinates), zoom);

    map.on('click', function (e) {

      if (marker !== null) {
        map.removeLayer(marker);
      }
      marker = L.marker([e.latlng.lat, e.latlng.lng])
        .addTo(map)
        .on('click', function (e) {
          marker.bindPopup('Latitude: ' + e.latlng.lat + '<br> Longitude: ' + e.latlng.lng).openPopup();
        });

      self.setValue(e.latlng.lat, e.latlng.lng);

    });

  },
}
</script>
<style>
.leaflet-control-layers-toggle {
  background-image: url("data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABoAAAAaCAQAAAADQ4RFAAACf0lEQVR4AY1UM3gkARTePdvdoTxXKc+qTl3aU5U6b2Kbkz3Gtq3Zw6ziLGNPzrYx7946Tr6/ee/XeCQ4D3ykPtL5tHno4n0d/h3+xfuWHGLX81cn7r0iTNzjr7LrlxCqPtkbTQEHeqOrTy4Yyt3VCi/IOB0v7rVC7q45Q3Gr5K6jt+3Gl5nCoDD4MtO+j96Wu8atmhGqcNGHObuf8OM/x3AMx38+4Z2sPqzCxRFK2aF2e5Jol56XTLyggAMTL56XOMoS1W4pOyjUcGGQdZxU6qRh7B9Zp+PfpOFlqt0zyDZckPi1ttmIp03jX8gyJ8a/PG2yutpS/Vol7peZIbZcKBAEEheEIAgFbDkz5H6Zrkm2hVWGiXKiF4Ycw0RWKdtC16Q7qe3X4iOMxruonzegJzWaXFrU9utOSsLUmrc0YjeWYjCW4PDMADElpJSSQ0vQvA1Tm6/JlKnqFs1EGyZiFCqnRZTEJJJiKRYzVYzJck2Rm6P4iH+cmSY0YzimYa8l0EtTODFWhcMIMVqdsI2uiTvKmTisIDHJ3od5GILVhBCarCfVRmo4uTjkhrhzkiBV7SsaqS+TzrzM1qpGGUFt28pIySQHR6h7F6KSwGWm97ay+Z+ZqMcEjEWebE7wxCSQwpkhJqoZA5ivCdZDjJepuJ9IQjGGUmuXJdBFUygxVqVsxFsLMbDe8ZbDYVCGKxs+W080max1hFCarCfV+C1KATwcnvE9gRRuMP2prdbWGowm1KB1y+zwMMENkM755cJ2yPDtqhTI6ED1M/82yIDtC/4j4BijjeObflpO9I9MwXTCsSX8jWAFeHr05WoLTJ5G8IQVS/7vwR6ohirYM7f6HzYpogfS3R2OAAAAAElFTkSuQmCC") !important;
}

.leaflet-popup-content-wrapper {
  border-radius: 1px;
}

.leaflet-top.leaflet-left .leaflet-control-zoom {
  box-shadow: 0 0 7px #999 !important;
}

#map {
  z-index: 0;
}
</style>
