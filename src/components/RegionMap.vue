<template>
  <div>
    <div ref="map" v-show="mapReady" class="map"></div>
    <button @click="editMode = true" v-if="!editMode">Edit</button>
    <button v-if="editMode" @click="saveChanges">Save</button>
    <button v-if="editMode" @click="cancelChanges">Cancel</button>
  </div>
</template>

<style lang="scss" scoped>
.map {
  width: 80vw;
  height: 80vh;
}
</style>

<script lang="ts">
import { Vue, Component, Prop, Watch } from 'vue-property-decorator';

export interface LatLng {
  Lat: number;
  Lng: number;
}

export interface Area {
  AreaCode: string;
  OutCodes: string[];
}

@Component({})
export default class MapsTest extends Vue {
  protected get internalAreaStrings(): string[] {
    return this.internalAreas.map(ia => ia.AreaCode);
  }
  protected get internalOutCodeStrings(): string[] {
    return this.internalAreas.reduce(
      (p, c) => p.concat(...c.OutCodes),
      [] as string[]
    );
  }

  @Prop({ default: [] }) public value!: Area[];
  private outcodesByArea: any[] = [];
  private center: LatLng = { Lat: 54.5, Lng: -3 };
  private map!: google.maps.Map;
  private mapReady: boolean = false;
  private currentFeature!: google.maps.Data.Feature;
  private currentOutCodes!: google.maps.Data.Feature[];
  private internalAreas: Area[] = [];
  private editMode: boolean = false;

  @Watch('value', { deep: true })
  protected updateInternalValues() {
    this.internalAreas = JSON.parse(JSON.stringify(this.value));

    if (this.mapReady) {
      this.updateRenderedAreas();
    }
  }

  protected getGoogle() {
    return (window as any).google;
  }

  protected created() {
    this.updateInternalValues();
  }

  protected async mounted() {
    const response = await fetch('/mapdata/outcodes-by-area.json');
    this.outcodesByArea = await response.json();

    await this.addMapsScript();
  }

  protected async addMapsScript() {
    try {
      const apiKey = '';
      const mapsUrl = `https://maps.googleapis.com/maps/api/js?key=${apiKey}&libraries=places`;
      if (!document.querySelectorAll(`[src="${mapsUrl}"]`).length) {
        const mapScriptElement = document.createElement('script');
        mapScriptElement.type = 'text/javascript';
        mapScriptElement.src = mapsUrl;
        mapScriptElement.onload = () => this.initialiseMap();
        document.body.appendChild(mapScriptElement);
      } else {
        await this.initialiseMap();
      }
    } catch (ex) {
      this.mapReady = false;
    }
  }

  protected async initialiseMap() {
    const el = this.$refs.map as HTMLElement;
    this.map = new (this.getGoogle()).maps.Map(el, {
      center: new (this.getGoogle()).maps.LatLng(
        this.center.Lat,
        this.center.Lng
      ),
      zoom: 5,
      minZoom: 5,
      maxZoom: 15,
      mapTypeId: this.getGoogle().maps.MapTypeId.ROADMAP,
      mapTypeControl: false,
      streetViewControl: false,
      clickableIcons: false
    });

    this.mapReady = true;

    await this.loadAreas();
    this.updateRenderedAreas();
  }

  protected updateRenderedAreas(featureSet?: google.maps.Data.Feature[]) {
    let itterateOn: google.maps.Data | google.maps.Data.Feature[];
    if (featureSet) {
      itterateOn = featureSet;
    } else {
      itterateOn = this.map.data;
    }

    itterateOn.forEach(feature => {
      const outCode = feature.getProperty('o');
      if (outCode) {
        if (this.outCodeIsSelected(outCode)) {
          this.map.data.overrideStyle(feature, { fillColor: 'green' });
        }
      } else {
        const areaCode = feature.getProperty('a');
        if (areaCode) {
          if (this.areaIsSelected(areaCode)) {
            if (this.areaHasFullCoverage(areaCode)) {
              this.map.data.overrideStyle(feature, { fillColor: 'green' });
            } else {
              this.map.data.overrideStyle(feature, { fillColor: 'yellow' });
            }
          } else {
            this.map.data.revertStyle(feature);
          }
        }
      }
    });
  }

  protected areaIsSelected(areaCode: string): boolean {
    return this.internalAreaStrings.indexOf(areaCode) !== -1;
  }

  protected areaHasFullCoverage(areaCode: string): boolean {
    const expectedOutcodes = this.outcodesByArea.find(
      a => a.areaCode === areaCode
    ).outcodes as string[];
    return expectedOutcodes.reduce(
      (p, c) => p && this.outCodeIsSelected(c),
      true as boolean
    );
  }

  protected outCodeIsSelected(outCode: string): boolean {
    return this.internalOutCodeStrings.indexOf(outCode) !== -1;
  }

  protected async loadAreas() {
    await new Promise(r =>
      this.map.data.loadGeoJson('/mapdata/areas.json', undefined, () => r())
    );

    this.map.data.addListener('click', this.featureClickedHandler);
  }

  protected runFunctionOnMatchingFeatures(
    predicate: (feature: google.maps.Data.Feature) => true | false,
    fun: (feature: google.maps.Data.Feature) => void
  ) {
    this.map.data.forEach(f => {
      if (predicate(f)) {
        fun(f);
      }
    });
  }

  protected featureClickedHandler(event: any) {
    if (
      this.currentFeature &&
      !event.feature.getProperty('o') &&
      this.currentFeature !== event.feature
    ) {
      // if we have a current feature, the event feature isn't an outcode
      // and the current feature isn't the event feature
      // clear outcodes
      this.currentOutCodes.forEach(f => this.map.data.remove(f));
      // add region back
      this.map.data.add(this.currentFeature);
      // update the area incase it was selected and now isn't or vice versa
      this.updateRenderedAreas([this.currentFeature]);
    }

    if (!event.feature.getProperty('o')) {
      // handle area selection
      this.currentFeature = event.feature;
      // clear data and load next geo data
      this.map.data.remove(event.feature);
      this.map.data.loadGeoJson(
        `/mapdata/${this.currentFeature.getProperty('a')}-geo.json`,
        undefined,
        features => {
          this.currentOutCodes = features;
          this.updateRenderedAreas(features);
        }
      );
    } else {
      if (!this.editMode) {
        return;
      }
      this.selectOutCode(event.feature);
    }
  }

  protected selectOutCode(feature: google.maps.Data.Feature) {
    // is the feature already selected?
    const areaCode = feature.getProperty('a');
    const outCode = feature.getProperty('o');
    if (this.outCodeIsSelected(outCode)) {
      // a few features have multiple polygons
      // we need to look for any with a matching outcode property
      // and clear that feature
      this.runFunctionOnMatchingFeatures(
        f => f.getProperty('o') === outCode,
        f => this.map.data.revertStyle(f)
      );

      // remove outcode from selected areas
      const area = this.internalAreas.find(a => a.AreaCode === areaCode)!;
      area.OutCodes.splice(area.OutCodes.indexOf(outCode), 1);
      // if area has no outcodes left, remove area
      if (area.OutCodes.length === 0) {
        this.internalAreas.splice(this.internalAreas.indexOf(area), 1);
      }
    } else {
      // select the outcode
      this.runFunctionOnMatchingFeatures(
        f => f.getProperty('o') === outCode,
        f => this.map.data.overrideStyle(f, { fillColor: 'green' })
      );
      // look for the area and add it if we don't have it
      const area = this.internalAreas.find(a => a.AreaCode === areaCode);
      if (!area) {
        this.internalAreas.push({
          AreaCode: areaCode,
          OutCodes: [outCode]
        });
      } else {
        area.OutCodes.push(outCode);
      }
    }
  }

  protected saveChanges() {
    this.$emit('input', this.internalAreas);
    this.editMode = false;
  }

  protected cancelChanges() {
    this.updateInternalValues();
    this.editMode = false;
  }
}
</script>
