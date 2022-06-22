<template>
  <div class="font-sans grid grid-cols-2 max-w-6xl mx-auto m-6">
    <form class="flex flex-col space-y-6 rounded bg-gray-50 p-6 shadow-md">

      <h1 class="text-2xl font-bold">Input Data</h1>

      <label class="block">
        <span class="sr-only">Upload Glucose Data</span>
        <input type="file" accept=".csv" @change="handleFileUpload($event)" class="block w-full text-sm text-slate-500
              file:mr-4 file:py-2 file:px-4
              file:rounded-full file:border-0
              file:text-sm file:font-semibold
              file:bg-violet-50 file:text-violet-700
              hover:file:bg-violet-100
      " />
      </label>

      <ul>
        <li class="cursor-pointer hover:bg-gray-100" v-for="note in notes" :key="note.time"
          @click="selectedDate = note.time; changeDate(); title = note.note;">
          <strong>{{ note.time }}</strong>: {{ note.note }}
        </li>
      </ul>

      <label class="block">
        <span class="block text-sm font-medium text-slate-700">Title</span>
        <input v-model="title"
          class="border rounded border-slate-400 placeholder-slate-400 contrast-more:border-slate-400 contrast-more:placeholder-slate-500" />
      </label>

      <label class="block">
        <span class="block text-sm font-medium text-slate-700">Select start date/time:</span>
        <select v-model="selectedDate" @change="changeDate()"
          class="border rounded border-slate-400 placeholder-slate-400 contrast-more:border-slate-400 contrast-more:placeholder-slate-500">
          <option v-for="date in dates" :key="date">
            {{ date }}
          </option>
        </select>
      </label> {{ selectedDate }}
    </form>

    <div style="width: 512px;" class="p-16">
      <h1 class="text-xl text-black text-center font-bold">{{ title }}</h1>

      <div class="mx-auto">
        <canvas id="myChart" ref="myChart"></canvas>
      </div>
    </div>

  </div>
</template>

<script>
import Chart from 'chart.js/auto';
import 'chartjs-adapter-date-fns';
import annotationPlugin from 'chartjs-plugin-annotation';
import Papa from 'papaparse';
import parse from 'date-fns/parse'
import subHours from 'date-fns/subHours'
import addHours from 'date-fns/addHours'
import isAfter from 'date-fns/isAfter'
import isBefore from 'date-fns/isBefore'
import differenceInHours from 'date-fns/differenceInHours'
import differenceInMinutes from 'date-fns/differenceInMinutes/index.js';
import intervalToDuration from 'date-fns/intervalToDuration'
import format from 'date-fns/format'

Chart.register(annotationPlugin);

function pad(num, size) {
  num = num.toString();
  while (num.length < size) num = "0" + num;
  return num;
}

export default {
  name: 'IndexPage',

  data() {
    return {
      title: 'Glucose after ..',
      file: '',
      content: [],
      parsed: false,
      dates: [],
      data: [],
      notes: [],
      selectedDate: null,
      subset: [],
      chart: null,
    }
  },

  methods: {
    handleFileUpload(event) {
      this.file = event.target.files[0];
      this.parseFile();
    },

    parseFile() {

      const reader = new FileReader();
      if (this.file.name.includes(".csv")) {
        reader.onload = (res) => {
          let content = res.target.result;

          // remove first line
          content = content.substring(content.indexOf("\n") + 1)

          Papa.parse(content, {
            header: true,
            skipEmptyLines: true,
            complete: function (results) {
              this.content = results;
              this.parsed = true;

              const sortedData = results.data.sort((a, b) => {
                return parse(a["Device Timestamp"], 'dd-MM-yyyy H:mm', new Date) - parse(b["Device Timestamp"], 'dd-MM-yyyy H:mm', new Date)
              })

              console.log(sortedData);

              var data = [];
              var notes = [];
              sortedData.forEach(item => {
                if (item['Historic Glucose mmol/L'])
                  data.push({ time: item['Device Timestamp'], y: item['Historic Glucose mmol/L'] * 18, });
                if (item['Scan Glucose mmol/L'])
                  data.push({ time: item['Device Timestamp'], y: item['Scan Glucose mmol/L'] * 18, });
                if (item['Notes'])
                  notes.push({ time: item['Device Timestamp'], note: item['Notes'], });
              });

              this.dataset = data;
              this.notes = notes;

              var dates = this.dataset.map(el => el.time);
              dates = new Set(dates);
              this.dates = dates;

              console.log(this.dataset)
            }.bind(this)
          });

        };
        reader.onerror = (err) => console.log(err);
        reader.readAsText(this.file);
      }

    },

    changeDate() {
      console.log('changeDate()');
      var referenceDate = parse(this.selectedDate, 'dd-MM-yyyy H:mm', new Date)
      var startDate = subHours(parse(this.selectedDate, 'dd-MM-yyyy H:mm', new Date), 1)
      var endDate = addHours(parse(this.selectedDate, 'dd-MM-yyyy H:mm', new Date), 4)
      var subset = this.dataset.filter(function (item) {
        var date = parse(item['time'], 'dd-MM-yyyy H:mm', new Date)

        if (!isAfter(date, endDate) && !isBefore(date, startDate)) {
          return true;
        }

        return false;
      });

      console.log(subset.length);
      subset.map(function (item) {
        var date = parse(item['time'], 'dd-MM-yyyy H:mm', new Date)
        // item['x'] = (isAfter(date, referenceDate) ? '' : '-') + differenceInMinutes(date, referenceDate)
        item['x'] = format(date, "H:mm")
        const inverval = intervalToDuration({
          start: startDate,
          end: date
        })
        item['x'] = pad(inverval.hours, 2) + ":" + pad(inverval.minutes, 2)
      });

      console.log(subset);
      this.subset = subset;

      this.renderChart();
    },

    renderChart() {
      // console.log(this.dataset);
      // console.log(demodata.filter(x => x.y > 0))
      var henkDerp = this.subset;
      const ctx = this.$refs.myChart;
      // if(this.chart != null){
      //   this.chart.destroy();
      //   this.chart = null;
      // }


      let chartStatus = Chart.getChart("myChart"); // <canvas> id
      if (chartStatus != undefined) {
        chartStatus.destroy();
      }

      new Chart(ctx, {
        data: {
          datasets: [{
            data: henkDerp, //.filter(x => x.y > 0),
            label: "mm/dl",
            // backgroundColor: gradientStroke,
            // fillColor: gradientStroke,
            borderColor: '#000',
            strokeColor: "#ff6c23",
            pointColor: "#fff",
            pointStrokeColor: "#ff6c23",
            pointHighlightFill: "#fff",
            pointHighlightStroke: "#ff6c23",
          }]
        },
        type: "line",
        options: {
          spanGaps: true,
          tension: 0.3,
          aspectRatio: 1,
          scales: {
            x: {
              // type: "linear",
              type: 'time',
              //     type: 'time',
              //     parsing: false,

              time: {
                parser: 'HH:mm',
                unit: 'hour',
                displayFormats: {
                  hour: 'HH:mm'
                },
              },

              ticks: {
                // Include a dollar sign in the ticks
                callback: function (value, index, ticks) {
                  return (index > 0 ? '+' : '') + index;
                }
              }

            }
          },

          plugins: {
            legend: { display: false },
            annotation: {
              annotations: {
                box1: {
                  type: "box",
                  yMin: 70,
                  yMax: 100,
                  backgroundColor: "rgba(0, 200, 0, 0.3)",
                  borderColor: "rgba(0,0,0,0)",
                },
                box2: {
                  type: "box",
                  yMin: 100,
                  yMax: 130,
                  backgroundColor: "rgba(0, 200, 0, 0.1)",
                  borderColor: "rgba(0,0,0,0)",
                },
                box3: {
                  type: "box",
                  yMin: 130,
                  yMax: 160,
                  backgroundColor: "rgba(200, 0, 0, 0.1)",
                  borderColor: "rgba(0,0,0,0)",
                },
                line1: {
                  type: 'line',
                  xMin: "01:00",
                  xMax: "01:00",
                  borderColor: 'rgb(255, 99, 132)',
                  borderWidth: 2,
                },
                line1: {
                  type: 'line',
                  xMin: "01:00",
                  xMax: "01:00",
                  borderColor: 'rgb(255, 99, 132)',
                  borderWidth: 2,
                },
                label1: {
                  type: 'label',
                  position: 'start',
                  content: ['Eating time'],
                  xValue: "01:00",
                  yValue: 160,
                  backgroundColor: 'rgba(255, 99, 132)',
                  color: 'rgba(255, 255, 255)',
                  font: {
                    size: 12
                  }
                },
              }
            }
          },
        },

        plugins: [annotationPlugin]

      });
    }
  },

  mounted() {
  }
}
</script>
