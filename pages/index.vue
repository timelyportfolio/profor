<template :fulldata="fulldata">
  <div class="container-fluid" style="margin-top: 2em;">
    <div class="col-md-3" style="overflow:auto; min-height: 100px; max-height:400px; position:fixed; top:150px;">
      <h4>Filters</h4>
      <filters  v-on:checked-nodes="checkHandler"></filters>
    </div>
    <div class="col-md-9 ml-md-auto" style="padding-left: 50px;">
      <div class="row align-items-start justify-content-center" style="margin-top:2em;">
        <VegaGeomap
          :spec = "spec_geo"
          :use-viewbox = "true"
          :use-tooltip = "true"
          :tooltip-options = "{
            showAllFields:false,
            fields: [
              {
                field: 'region',
                title: 'Region'
              },
              {
                field: 'subregion',
                title: 'Subregion'
              },
              {
                field: 'country',
                title: 'Country'
              },
              {
                field: 'size',
                title: 'ArticleCount'
              }
            ]
          }"
        >
        </VegaGeomap>
      </div>
      <div class="row align-items-start" style="margin-top:2em;">
        <div class="col col-md-12">
          <h5>Intervention by Region</h5>
          <VegaBarFacet
            :matrix = "matrix_geoint"
            x = "ArticleCount"
            y = "Intervention"
            facet = "Region"
          >
          </VegaBarFacet>
        </div>
      </div>
      <div class="row align-items-start" style="margin-top:2em;">
        <div class="col col-md-12">
          <h5>Habitat by Region</h5>
          <VegaBarFacet
            :matrix = "matrix_geohab"
            x = "ArticleCount"
            y = "Habitat"
            facet = "Region"
          >
          </VegaBarFacet>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
  import axios from 'axios'
  import {arrayeq} from '~/assets/utils.js'

  import {set, nest} from 'd3-collection'
  import {merge} from 'd3-array'

  import Filters from '~/components/Filters.vue'
  import VegaGeomap from '~/components/VegaGeomap.vue'
  import VegaBarFacet from '~/components/VegaBarFacet.vue'

  import Codes from '../static/codes.js'

  export default {
    components: {
      Filters,
      VegaGeomap,
      VegaBarFacet
    },
    created: function() {
      axios.get('articles_profor.json').then(response => {
        this.articles = response.data
      })
      .catch(e => {
        console.log('error getting data', e)
        //this.errors.push(e)
      })
      axios.get('data_profor.json').then(response => {
        this.fulldata = response.data
        window.fulldata = response.data
      })
      .catch(e => {
        console.log('error getting data', e)
        //this.errors.push(e)
      })
    },
    data: function() {
      return {
        articles: [],
        fulldata: [],
        checkedfilters: []
      }
    },
    computed: {
      filtered: function() {
        //if(this.checkedgeo.length > 0) {
          return {
            data: this.filterData(this.checkedfilters),
            filters_geo: this.checkedfilters.filter(dd=>dd.type==='geo').map(dd=>dd.name),
            filters_int: this.checkedfilters.filter(dd=>dd.type==='intervention').map(dd=>dd.type_code),
            filters_hab: this.checkedfilters.filter(dd=>dd.type==='habitat').map(dd=>dd.code)
          }
        //}
      },
      spec_geo: function() {
        var filtered = this.filtered
        var data = filtered.data
        var geo = filtered.filters_geo
        var filtered_geo = []
        data.forEach(function(d) {
          d.geo.forEach(function(dd) {
            if(geo.indexOf(dd.subregion) > -1) {
              filtered_geo.push({
                id: dd.id,
                region: dd.region,
                subregion: dd.subregion,
                country: dd['Study_country.x'],
                aid: d.aid
              })
            }
          })
        })

        var nested = nest()
          .key(d=>d.id)
          .rollup(d=>{return {
            id: d[0].id,
            region: d[0].region,
            subregion: d[0].subregion,
            country: d[0].country,
            size: set(d.map(dd=>dd.aid)).size()
          }})
          .entries(filtered_geo);
        return {        
          "$schema": "https://vega.github.io/schema/vega/v3.0.json",
          "width": 600,
          "height": 500,
          "autosize": "fit",
          "background": "ded",

          "signals": [
            { "name": "tx", "update": "width / 2"},
            { "name": "ty", "update": "height / 2"},
            {
              "name": "scale",
              "value": 100,
              "on": [{
                "events": {"type": "wheel", "consume": true},
                "update": "clamp(scale * pow(1.0005, -event.deltaY * pow(16, event.deltaMode)), 100, 3000)"
              }]
            },
            {
              "name": "angles",
              "value": [0, 0],
              "on": [{
                "events": "mousedown",
                "update": "[rotateX, centerY]"
              }]
            },
            {
              "name": "cloned",
              "value": null,
              "on": [{
                "events": "mousedown",
                "update": "copy('projection')"
              }]
            },
            {
              "name": "start",
              "value": null,
              "on": [{
                "events": "mousedown",
                "update": "invert(cloned, xy())"
              }]
            },
            {
              "name": "drag", "value": null,
              "on": [{
                "events": "[mousedown, window:mouseup] > window:mousemove",
                "update": "invert(cloned, xy())"
              }]
            },
            {
              "name": "delta", "value": null,
              "on": [{
                "events": {"signal": "drag"},
                "update": "[drag[0] - start[0], start[1] - drag[1]]"
              }]
            },
            {
              "name": "rotateX", "value": 0,
              "on": [{
                "events": {"signal": "delta"},
                "update": "angles[0] + delta[0]"
              }]
            },
            {
              "name": "centerY", "value": 0,
              "on": [{
                "events": {"signal": "delta"},
                "update": "clamp(angles[1] + delta[1], -60, 60)"
              }]
            },

            { "name": "borderWidth", "value": 1 },
            { "name": "invert", "value": false }
          ],

          "projections": [
            {
              "name": "projection",
              "type": "mercator",
              "scale": {"signal": "scale"},
              "rotate": [{"signal": "rotateX"}, 0, 0],
              "center": [0, {"signal": "centerY"}],
              "translate": [{"signal": "tx"}, {"signal": "ty"}]
            }
          ],

          "data": [
            {
              "name": "geosum",
              "values": nested.map(d=>d.value)
            },
            {
              "name": "world",
              "url": "world-110m.json",
              "format": {
                "type": "topojson",
                "feature": "countries"
              },
              "transform": [
                { 
                  "type": "lookup", "from": "geosum", "key": "id",
                  "fields": ["id"], "values": ["region", "subregion", "country", "size"]
                }//,
                //{ "type": "filter", "expr": "datum.size != null" }
              ]
            },
          ],

          "scales": [
            {
              "name": "color",
              "type": "sequential",
              "domain": {"data": "geosum", "field": "size"},
              "range": "heatmap",
              "zero": true
            }
          ],

          "marks": [
            {
              "type": "shape",
              "from": {"data": "world"},
              "encode": {
                "update": {
                  "strokeWidth": {"signal": "+borderWidth"},
                  "stroke": {"signal": "invert ? '#777' : '#bbb'"},
                  "fill": [
                    {
                      "test": "datum.size!==null",
                      "scale": "color",
                      "field": "size"
                    },
                    {"value": "lightgray"}
                  ],
                  "zindex": {"value": 0}
                },
                "hover": {
                  "strokeWidth": {"signal": "+borderWidth + 1"},
                  "stroke": {"value": "firebrick"},
                  "zindex": {"value": 1}
                }
              },
              "transform": [
                { "type": "geoshape", "projection": "projection" }
              ]
            }
          ],

          "legends": [
            {
              "fill": "color",
              "type": "gradient",
              "orient": "top-left",
              "title": "Count of Articles",
              "format": ",.0f"
            }
          ],

        }
      },
      matrix_geoint: function() {
        var filtered = this.filtered
        var data = filtered.data
        var geo = filtered.filters_geo
        var int = filtered.filters_int
        var filtered_geoint = []
        data.forEach(function(d) {
          d.geo.forEach(function(dd) {
            if(geo.indexOf(dd.subregion) > -1) {
              d.intervention.forEach(function(ddd){
                if(int.indexOf(ddd.Int_type) >  -1) {
                  var intervention = Codes().filter(dc => dc.code === ddd.Int_type)[0]
                  filtered_geoint.push({
                    region: dd.region,
                    subregion: dd.subregion,
                    country: dd['Study_country.x'],
                    intervention: intervention ? intervention.code_def : ddd.Int_type,
                    aid: d.aid
                  })
                }
              })
            }
          })
        })

        var nested = nest()
          .key(d=>d.region)
          .key(d=>d.intervention)
          .rollup(d=>{ return {
            Region: d[0].region,
            Intervention: d[0].intervention,
            //Description: Codes().filter(dc => dc.code=== d[0].int_group)[0].code_def,
            ArticleCount: set(d.map(dd=>dd.aid)).size()
          }})
          .entries(filtered_geoint)
          .map(d=>d.values.map(dd=>dd.value));
        return merge(nested)
      },
      matrix_geohab: function() {
        var filtered = this.filtered
        var data = filtered.data
        var geo = filtered.filters_geo
        var hab = filtered.filters_hab
        var filtered_geohab = []
        data.forEach(function(d) {
          d.geo.forEach(function(dd) {
            if(geo.indexOf(dd.subregion) > -1) {
              d.habitat.forEach(function(ddd){
                if(hab.indexOf(ddd['Biome.']) >  -1) {
                  var habitat = Codes().filter(dc => dc.code === ddd['Biome.'])[0]
                  filtered_geohab.push({
                    region: dd.region,
                    subregion: dd.subregion,
                    country: dd['Study_country.x'],
                    habitat: habitat ? habitat.code_def : ddd['Biome.'],
                    aid: d.aid
                  })
                }
              })
            }
          })
        })

        var nested = nest()
          .key(d=>d.region)
          .key(d=>d.habitat)
          .rollup(d=>{ return {
            Region: d[0].region,
            Habitat: d[0].habitat,
            //Description: Codes().filter(dc => dc.code=== d[0].int_group)[0].code_def,
            ArticleCount: set(d.map(dd=>dd.aid)).size()
          }})
          .entries(filtered_geohab)
          .map(d=>d.values.map(dd=>dd.value));
        return merge(nested)
      }
    },
    methods: {
      filterData: function(filters) {
        var geo = filters.filter(dd=>dd.type==='geo').map(dd=>dd.name); 
        var habitat = filters.filter(dd=>dd.type==='habitat').map(dd=>dd.code);
        var intervention = filters.filter(dd=>dd.type==='intervention').map(dd=>dd.type_code);
        var outcome = filters.filter(dd=>dd.type==='outcome').map(dd=>dd.code);

        return this.fulldata
          .filter(
            function(d) {
              var foundgeo =  d.geo.some(function(dd) {
                return geo.indexOf(dd.subregion) > -1
              })
              var foundhab = d.habitat.some(function(dd) {
                return habitat.indexOf(dd['Biome.']) > -1
              })
              var foundint = d.intervention.some(function(dd) {
                return intervention.indexOf(dd['Int_type']) > -1
              })
              var foundout = d.outcome.some(function(dd) {
                return outcome.indexOf(dd['Out_subtype']) > -1
              })
              return foundgeo && foundhab && foundint && foundout
                /*
                intervention.indexOf(d.Int_type) > -1 &&
                outcome.indexOf(d.Outcome) > -1
                */
            }
          )
      },
      checkHandler: function(checkednodes) {
        var allfilters = [].concat(checkednodes.filter(d=>d.colname==="subregion").map(d=>{return {type:'geo','id':d.id,'name':d.name}}))
          .concat(checkednodes.filter(d=>d.colname==="ecoregion").map(d=>{return {type:'habitat','id':d.id,'code':d.code}}))
          .concat(checkednodes.filter(d=>d.colname==="type").map(d=>{return {type:'intervention','id':d.id,'type_code':d.type_code}}))
          .concat(checkednodes.filter(d=>d.colname==="outcome").map(d=>{return {type:'outcome','id':d.id,'code':d.code}}))
        
        if(!arrayeq(this.checkedfilters, allfilters, function(d){return d.id})) {
          this.checkedfilters = allfilters
        }
      }
    }
  }
</script>

<style>
  .vg-tooltip {
    visibility: hidden;
    padding: 6px;
    border-radius: 3px;
    position: fixed;
    z-index: 2000;
    font-family: sans-serif;
    font-size: 11px;

    /* The default look of the tooltip is the same as .light-theme
    but it can be overwritten by .dark-theme */
    background-color: rgba(255, 255, 255, 0.9);
    border: 1px solid #d9d9d9;
    color: black;
  }
  .vg-tooltip td.key, .vg-tooltip td.value {
    overflow: hidden;
    text-overflow: ellipsis;
  }
  .vg-tooltip td.key {
    color: #808080;
    max-width: 150px;
    text-align: right;
    padding-right: 1px;
  }
  .vg-tooltip td.value {
    max-width: 200px;
    text-align: left;
  }

  /* Dark and light color themes */
  .vg-tooltip.dark-theme {
    background-color: rgba(64, 64, 64, 0.9);
    border: none;
    color: white;
  }
  .vg-tooltip.dark-theme td.key {
    color: #bfbfbf;
  }
  .vg-tooltip.light-theme {
    background-color: rgba(255, 255, 255, 0.9);
    border: 1px solid #d9d9d9;
    color: black;
  }
  .vg-tooltip.light-theme td.key {
    color: #808080;
  }
</style>
