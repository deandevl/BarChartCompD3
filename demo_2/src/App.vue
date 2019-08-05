<template>
  <div class='demo_container'>
    <button v-on:click="plot_chart_clicked">Plot Bar Chart</button>
    <bar-chart-comp-d3
        title_1="Kentucky Coal Production"
        title_2="(1960 to 2010 in tons)"
        :axis="axis"
        :chart_data="chart_data"
        :margin_bottom="margin_bottom"
        :css_variables="css_variables">
      <svg class="svg_chart_1" width="1200" height="700"></svg>
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
        axis: {
          x: {
            title: 'Years',
            key: 'year',
            rotation: -65
          },
          y: {
            title: 'Coal Production',
            tic_format: '~s',
            groups: [
              {key: 'tons', format: '.3s', color: '#ff8c00'}
            ]
          }
        },
        margin_bottom: 75,
        css_variables: {
          bar_chart_compD3_color: 'white',
          bar_chart_compD3_background_color: 'black',
          bar_chart_compD3_axis_font_size: '12px',
          bar_chart_compD3_axis_color: 'white',
          bar_chart_compD3_rect_fill_opacity: '0.7',
          bar_chart_compD3_rect_stroke: 'none',
          bar_chart_compD3_tooltip_fill: 'white',

          input_comp_heading_color: 'white',
          input_comp_heading_font_size: '12px',
          input_comp_input_color: 'white',
          input_comp_input_border_color: 'white',
          input_comp_input_placeholder_color: 'white',
          input_comp_input_focus_background: 'black',

          button_comp_font_size: '12px'
        }
      };
    },
    components: {
      BarChartCompD3
    },
    methods: {
      plot_chart_clicked: function(){
        const convert = [
          {field: 'year',type: 'band'},
          {field: 'tons',type: 'linear'}
        ];
        const get_data = async () => {
          try {
            this.chart_data = await read_csv('data/kentucky_coal.csv',convert);
            //debug
            console.log(JSON.stringify(this.chart_data));
          }catch(e){
            console.log(e);
          }
        };
        get_data();
      }
    }
  }
</script>