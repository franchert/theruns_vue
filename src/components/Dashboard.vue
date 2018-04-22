<template>
  <div>
    <v-data-table :headers="headers" :items="runs" hide-actions>
      <template slot="items" slot-scope="r">
        <td><a :href="`https://www.strava.com/activities/${r.item.id }`" target="_blank">{{ r.item.id }}</a></td>
        <td>{{ r.item.date | moment('L') }}</td>
        <td>{{ r.item.distance }}</td>
        <td>{{ r.item.time }}</td>
        <td>{{ r.item.score }}</td>
      </template>
    </v-data-table>
  </div>
</template>

<script>
/* eslint-disable */
import strava from 'strava-v3';
import moment from 'moment';
import stravaconfig from '../../data/strava_config.json';
import dbconfig from '../../data/firebase_config.json';
import Firebase from 'firebase'

let app = Firebase.initializeApp(dbconfig)
let db = app.database()
let runsRef = db.ref('runs')

let finished = false;

export default {
  name: 'Dashboard',
  data() {
    return {
      runs: [],
      nextPage: 0,
      newRun: {
        id: '',
        distance: '',
        elapsed_time: ''
      },
      headers: [
        { text: 'ID', value: 'id'},
        { text: 'date', value: 'date'},
        { text: 'distance', value: 'distance'},
        { text: 'time', value: 'time'},
        { text: 'score', value: 'score'},
      ]
    }
  },
  methods: {
    getRuns(pag) {
      strava.athlete.listActivities({'access_token': stravaconfig.access_token, page: pag}, 
      (err, payload, limits) => {
        if (!err) {
          if (payload.length > 0) {
            let newRuns = [];
            for (let i = 0; i < payload.length; i++) {
              let data = {
                  id: payload[i]['id'],
                  date: payload[i]['start_date_local'],
                  distance: Math.round((payload[i]['distance'] * 0.000621371192)*100)/100,
                  time: payload[i]['elapsed_time'],
                  score: Math.round(scorer(Math.round((payload[i]['distance'] * 0.000621371192)*100)/100, payload[i]['elapsed_time'])*100)/100
              };
              newRuns.push(data);
            }
            const combined = [...this.runs, ...newRuns];
            const temp = this.filterDuplicates(combined);
            const nextPage = pag + 1;
            console.log(this.runs.length, temp.length);
            if(this.runs.length != temp.length){
              this.getRuns(nextPage);
            }
            this.runs = temp;
          } else {
            runsRef.push(this.runs);
          }
        } else {
          console.log(err);
        }
      });
    },
    filterDuplicates(items) {
      const seen = {};
      return items.filter((item) => {
        if (Object.prototype.hasOwnProperty.call(seen, item.id)) {
          return false;
        }
        seen[item.id] = true;
        return true;
      });
    }
  },
  created() {
    Promise.all([
      // db.ref('nextpage').once('value'),
      db.ref('runs').once('value')
    ]).then((n) => {
      // let pagesData = n[0].val()
      // let pagesData_keys = Object.keys(pagesData);
      // this.nextPage = pagesData[pagesData_keys[pagesData_keys.length - 1]];
      // db.ref('nextpage').once("value", function(n) {
      //   if(n.numChildren() > 2){
      //     db.ref('nextpage').child(pagesData_keys[0]).remove();
      //   }
      // });
      if(n[0].val()){
        let runsData = n[0].val();
        let runsData_keys = Object.keys(runsData);
        this.runs = runsData[runsData_keys[runsData_keys.length - 1]];
        console.log(this.runs);
        db.ref('runs').once("value", function(n) {
          let numToDelete = n.numChildren() - 2;
          // console.log(numToDelete);
          for (let i = 0; i < numToDelete; i++){
            db.ref('runs').child(runsData_keys[0]).remove();
          }
        });
      } else {
        this.runs = [];
      }
      // console.log(this.nextPage, this.runs);
      this.getRuns(0);
    })
  },
  firebase: {
    runs: runsRef
  }
}
function scorer(distance, time) {
    const pace = ((time / 60) / distance) * 60;
    return ((Math.round(((400 * Math.pow(distance, 0.15)) - pace) * 100) / 100) + 50).toString().padStart(2,'0');;
}
</script>

<style>

</style>
