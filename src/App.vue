<template>
  <div id="app">
    <div class="title has-grey-background"><h1>Aleksey's Challenge</h1></div>
    <div id="user-input-container" class="user-input">
        <prism-editor :lineNumbers="true" v-model.trim="code" :code="code" language="js"></prism-editor>
        <div id="handle" class="fa fa-arrows-v"> </div>
    </div>
    <div id="chart-container" style="height:400px" class="chart">
      <apexchart width="100%" height="100%"  type="line" :options="options" :series="series"></apexchart>
    </div>
    <div class="footer has-grey-background">
      <button  @click="generateChart" class="generate-btn">Generate Chart</button>
    </div>
  </div>
</template>

<script>
import "prismjs";
import "prismjs/themes/prism-darkula.css";
import "vue-prism-editor/dist/VuePrismEditor.css"; 
import VueApexCharts from 'vue-apexcharts'
import PrismEditor from 'vue-prism-editor'
import JSON5 from 'json5' //For parsing Relaxed JSON strings, so we can use parse input without "" onto keys.
import interact from 'interactjs'
import moment from 'moment'

export default {
  name: 'app',
  data(){
    return {
      code: //Sample data
`{type: 'start', timestamp: 1519780251293, select: ['min_response_time', 'max_response_time'], group: ['os', 'browser']}
{type: 'span', timestamp: 1519780251293, begin: 1519780251001, end: 1519780251002}
{type: 'data', timestamp: 1519780251001, os: 'linux', browser: 'chrome', min_response_time: 0.1, max_response_time: 1.3}
{type: 'data', timestamp: 1519780251001, os: 'linux', browser: 'firefox', min_response_time: 0.1, max_response_time: 1.2}
{type: 'data', timestamp: 1519780251001, os: 'mac', browser: 'chrome', min_response_time: 0.2, max_response_time: 1.6}
{type: 'data', timestamp: 1519780251001, os: 'mac', browser: 'firefox', min_response_time: 0.3, max_response_time: 1.4}
{type: 'data', timestamp: 1519780251002, os: 'linux', browser: 'chrome', min_response_time: 0.2, max_response_time: 1.4}
{type: 'data', timestamp: 1519780251002, os: 'linux', browser: 'firefox', min_response_time: 0.3, max_response_time: 1.2}
{type: 'data', timestamp: 1519780251002, os: 'mac', browser: 'chrome', min_response_time: 0.4, max_response_time: 1.8}
{type: 'data', timestamp: 1519780251002, os: 'mac', browser: 'firefox', min_response_time: 0.5, max_response_time: 1.7}
{type: 'stop', timestamp: 1529780251293}`,
      options:{
        states: {
            hover: {
                filter: {
                    type: 'none',
                }
            },
        },
        selection: {
          enabled: false,
        },
        tooltip:{
          enabled:false
        },
        zoom:{
          enabled:false,
        },
        animations:{
          enabled:false
        },
        legend: {
          position: 'right'
        },        
        markers: {
          size: 6
        },
        xaxis: {
          labels: {
            type:'datetime',
            formatter: function(value, timestamp, index) {
              return moment(new Date(timestamp)).format("mm[:]ss")
            }
         }
      },
        chart: {
            toolbar: {
              show: false,
            },
        }

      },
      series: []
    }
  },
  created(){
    interact('#user-input-container').resizable({
      edges: {
        bottom:'#handle'
      }
    }).on('resizemove',event => {
    Object.assign(event.target.style, {
      height: `${event.rect.height}px`,
    })

    })
  },
  methods:{
    toCamelCase(str){
      return str.charAt(0).toUpperCase()+str.slice(1);
    },
    generateChart(){
      let parsedCode =  this.code.split('\n').map(JSON5.parse); //Spliting input by new lines;
      let foundAll = false;
      let foundSeries = [];
      while(!foundAll){
        let startPos = parsedCode.findIndex((element) => element.type == 'start');
        if(startPos == -1){ //No start element
          foundAll = true;
          break;
        }
        let endPos = parsedCode.findIndex((element) => element.type == 'stop');
        if(endPos == -1){ // If there is no stop event
          endPos = parsedCode.length;
          foundAll = true;
        }
        if(endPos == parsedCode.length + 1){ //If the stop event is the last input element;
          foundAll =true;
        }
        let numberOfEntries = endPos - startPos + 1;//Number of entries, for splicing
        foundSeries.push(parsedCode.splice(startPos,numberOfEntries));
      }
      if(foundSeries.length > 0){

        foundSeries.forEach(serie => { //If there is more than one series, it overrides the others
            this.series = [];
            let chartSeries = {}; //Using a object since js dont have name - value arrays;
            let begin,end; // Timestamps
            let startEvent = serie[0];//The start element is always the first one; 
            let span = serie.slice(0).reverse().find(el => el.type == 'span'); //Only the last span event of the series is relevant
            if(span){
              begin = span.begin;
              end = span.end;
            }
            let dataEvents = serie.filter(ser => {
              let bg,ed = true; //So if its empty, it does not try to filter;
              if(begin){
                bg =ser.timestamp >= begin?true:false;
              }
              if(end){
                ed = ser.timestamp <= end?true:false;
              }
              return  bg && ed  && ser.type == 'data';//Filter the data events
            });
            dataEvents.forEach(event => {
                startEvent.select.forEach(sel => {
                  let seriesName = '';
                  startEvent.group.forEach(grp => {
                      if(event[grp]){
                        seriesName += event[grp]+'_';
                      }
                  })
                 seriesName += sel;
                 if(!chartSeries.hasOwnProperty(seriesName)){
                   chartSeries[seriesName] = []; //If there is not a property with this name yet, set an array on this property
                 }
                  chartSeries[seriesName].push([event.timestamp,event[sel]]);
                })
            })
        for (var property in chartSeries) {
            if (chartSeries.hasOwnProperty(property)) { //So we dont get prototype properties
                let ser = {};
                ser.name = property.split('_').map(this.toCamelCase).join(" ");//Generating the label name
                ser.data = chartSeries[property];
                this.series.push(ser);
            }
        }
        });

      }else{
        alert('Error generating the chart!, does it have a start event?');
      }
    }
  },
  components: {
    'apexchart':VueApexCharts,
    PrismEditor
  }
}
</script>

<style>
 @import url('https://fonts.googleapis.com/css?family=Source+Sans+Pro&display=swap');
 @import url('https://fonts.googleapis.com/css?family=Source+Code+Pro&display=swap');
@import url('https://stackpath.bootstrapcdn.com/font-awesome/4.7.0/css/font-awesome.min.css');

#chart-container{
  touch-action:none;
  flex:1 1 auto;
  box-sizing: border-box;
  margin-bottom:63px;
}
#handle{
  background-color:black;
  color:#2E3440;
  font-size: 18px;
  padding:5px 10px;
  border-radius: 5px;
  text-align: center;
  position:absolute;
  left:calc(50% - 10px);
  bottom: -13px;
  z-index: 2;
}
#handle:hover{
  background-color:#2E3440;
  color:black;
}
#user-input-container{
  flex:1 1 auto;
  position: relative;
}
.title{
  padding:10px;
}
*{
  margin:0;
  padding:0;
  font-family: 'Source Sans Pro', sans-serif;
}
.prism-editor__line-numbers,.prism-editor__code{
  border-radius: 0px !important;
}
.chart{
  padding:20px;
}
.user-input{
  font-family: 'Source Code Pro', monospace;
}
#app{
  height: 100vh;
  display: flex;
  flex-flow:column nowrap;
  align-content: center;
}
.generate-btn{
  background-color:#017EFF;
  padding:10px 15px;
  font-size: 16px;
  text-transform: uppercase;
  border-radius: 3px;
  color:white;
  border:none;
  transition: all 0.3s ease;
  cursor: pointer;
}
.generate-btn:hover{
  color:#017EFF;
  background-color:white;
}
.has-grey-background{
  background-color:#dddee1;
}
.footer{
  padding:15px;
  position: fixed;
  bottom:0px;
  width:100%;
}
</style>
