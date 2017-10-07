<template :fulldata="fulldata">
  <div class="container-fluid" style="margin-top: 2em;">
    <div class="col col-3" style="overflow:auto; min-height: 100px; max-height:400px; position:fixed; top:150px;">
      <h4>Filters</h4>
      <filters  v-on:checked-nodes="checkHandler"></filters>
    </div>
  </div>
</template>

<script>
  import axios from 'axios'
  import {arrayeq} from '~/assets/utils.js'
  import Filters from '~/components/Filters.vue'

  export default {
    components: {
      Filters
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
          return this.filterData(this.checkedfilters)
        //}
      }
    },
    methods: {
      filterData: function(filters) {
        var geo = filters.filter(dd=>dd.type==='geo').map(dd=>dd.name); 
        // var habitat = filters.filter(dd=>dd.type==='habitat').map(dd=>dd.code);
        // var intervention = filters.filter(dd=>dd.type==='intervention').map(dd=>dd.type_code);
        // var outcome = filters.filter(dd=>dd.type==='outcome').map(dd=>dd.code);
      debugger  
        return this.fulldata
          .filter(
            function(d) {
              return d.geo.some(function(dd) {
                return geo.indexOf(dd.subregion) > -1;
              })
                /* &&
                habitat.indexOf(d["Biome."]) > -1 && 
                intervention.indexOf(d.Int_type) > -1 &&
                outcome.indexOf(d.Outcome) > -1
                */
            }
          )
      },
      checkHandler: function(checkednodes) {
        var allfilters = [].concat(checkednodes.filter(d=>d.colname==="subregion").map(d=>{return {type:'geo','id':d.id,'name':d.name}}))
          //.concat(checkednodes.filter(d=>d.colname==="ecoregion").map(d=>{return {type:'habitat','id':d.id,'code':d.code}}))
          //.concat(checkednodes.filter(d=>d.colname==="type").map(d=>{return {type:'intervention','id':d.id,'type_code':d.type_code}}))
          //.concat(checkednodes.filter(d=>d.colname==="outcome").map(d=>{return {type:'outcome','id':d.id,'code':d.code}}))
        
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
