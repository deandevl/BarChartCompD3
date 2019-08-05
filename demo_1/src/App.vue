<template>
  <div class='demo_container'>
    <section class="button_sec">
      <button v-on:click="stacked_chart_clicked">Plot Stacked Chart</button>
      <button v-on:click="unstacked_chart_clicked">Plot UnStacked Chart</button>
    </section>
    <bar-chart-comp-d3
        title_1="Population of States by Age Group"
        title_2="(Mike Bostock’s Block 3886208 Stacked Bar Chart)"
        :axis="axis"
        :stack_bars="stack_bars"
        :chart_data="chart_data"
        :css_variables="css_variables">
      <svg class="svgchart" width="1200" height="700"></svg>
    </bar-chart-comp-d3>
  </div>
</template>

<script>
  import BarChartCompD3 from 'barchartcompd3';
  import {read_csv} from 'd3fetchmodule';

  export default {
    name: 'App',
    data: function() {
      return {
        chart_data: null,
        stack_bars: null,
        axis: {
          x: {
            title: 'States',
            key: 'state',
            rotation: 0
          },
          y: {
            title: 'Population',
            tic_format: '~s',
            groups: [
              {key: 'Under 5 Years', format: '.3s', color: '#98abc5'},
              {key: '5 to 13 Years', format: '.3s', color: '#8a89a6'},
              {key: '14 to 17 Years', format: '.3s', color: '#7b6888'},
              {key: '18 to 24 Years', format: '.3s', color: '#6b486b'},
              {key: '25 to 44 Years', format: '.3s', color: '#a05d56'},
              {key: '45 to 64 Years', format: '.3s', color: '#d0743c'},
              {key: '65 Years and Over', format: '.3s', color: '#ff8c00'}
            ],
          }
        },
        convert: [
          {field: 'state',type: 'band'},
          {field: 'Under 5 Years',type: 'linear'},
          {field: '5 to 13 Years',type: 'linear'},
          {field: '14 to 17 Years',type: 'linear'},
          {field: '18 to 24 Years',type: 'linear'},
          {field: '25 to 44 Years',type: 'linear'},
          {field: '45 to 64 Years',type: 'linear'},
          {field: '65 Years and Over',type: 'linear'}
        ],
        css_variables: {
          bar_chart_compD3_rect_fill_opacity: '0.8',
          bar_chart_compD3_rect_stroke: 'none',
          bar_chart_compD3_axis_color: 'blue'
        }
      };
    },
    components: {
      BarChartCompD3
    },
    methods: {
      stacked_chart_clicked: function(){
        const get_data = async () => {
          try{
            this.stack_bars = true;
            //original data
            this.chart_data = await read_csv('data/data.csv', this.convert);
            //debug: data_test.csv -- has 'nice' values for testing
            //this.chart_data = await this.ioMixin_read_csv('data_test.csv',this.convert);
            //debug: console output
            console.log(JSON.stringify(this.chart_data));
          }catch(e){
            console.log(e);
          }
        };
        get_data();
      },
      unstacked_chart_clicked: function(){
        const get_data = async () => {
          try{
            this.stack_bars = false;
            //original data
            this.chart_data = await read_csv('data/data.csv', this.convert);
            //debug: data_test.csv -- has 'nice' values for testing
            //this.chart_data = await this.ioMixin_read_csv('data_test.csv',this.convert);
            //debug: console output
            //console.log(JSON.stringify(this.chart_data));
          }catch(e){
            console.log(e);
          }
        };
        get_data();
      }
    }
  }
</script>

<style>
  .button_sec {
    display: flex;
    flex-direction: row;
    justify-content: space-between;
    width: 300px;
  }
</style>