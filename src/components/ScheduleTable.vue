<template>
  <div class="row m-2">
    <div class="col-md-9 to-print data-div" ref="toPrint">
      <div class="row">
        <div class="col" v-for="row in rows" :key="`days_${row}`">
          <h4 v-text="row" class="d-flex justify-content-center"></h4>
          <hr>
          <div class="container weekday__items to-print" :style="{'height': `${dataHeight}px !important`}">
            <div class="row day__item mt-2" v-for="(item, i) in dataRows[row.toLowerCase()]"
                 :key="`weekday_item_${row.toLowerCase()}_${i}`">
              <div class="col-sm-3 text-center">
                <i :class="item.icon"></i>
              </div>
              <div class="col-sm-9 text-center">
                <span v-text="item.time.start"></span> - <span v-text="item.time.finish"></span>
              </div>
              <div class="col-sm-12">
                <span v-text="item.description"></span>
              </div>
              <div class="col-sm-12 d-flex justify-content-center no-print" v-if="!viewerMode">
                <div class="row">
                  <div class="col-sm-12">
                    <button class="btn btn-danger m-2" @click="deleteItem(row, i)">
                      <i class="bi bi-trash"></i>
                    </button>
                    <button class="btn btn-success m-2" @click="updateItem(row, i)">
                      <i class="bi bi-pencil"></i>
                    </button>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
    <div class="col-md-3 no-print">
      <div class="row" ref="maxHeight">
        <div class="col-sm-12 d-grid gap-2">
          <button class="btn mode__changer"
                  @click="viewerMode = !viewerMode"
                  :class="{'btn-primary' : viewerMode, 'btn-warning': !viewerMode}">
            <i class="bi" :class="{'bi-eye': !viewerMode, 'bi-shield-shaded': viewerMode}"
               v-text="viewerMode ? 'View as Admin' : 'View as Guest'"></i>
          </button>
        </div>
        <div class="col-sm-12 d-grid gap-2 mt-3">
          <button class="btn btn-primary" @click="printSchedule">
            <i class="bi bi-printer">Print</i>
          </button>
        </div>
        <div class="col-sm-12 text-center mt-3">
          <h4>Legend</h4>
        </div>
        <div class="col-sm-12">
          <ul>
            <li v-for="(icon, index) in icons" :key="`icon_${index}`">
              <i :class="icon.icon"></i> {{ icon.title }}
            </li>
          </ul>
        </div>
        <div class="col-sm-12" v-if="errors">
          <p
              class="text-danger error__list__item"
              v-for="(error, i) in errors"
              :key="`error_${i}`"
              v-text="error"
          ></p>
        </div>
        <div class="col-sm-12">
          <select class="form-control" v-model="form.day">
            <option :value="null" selected disabled>Choose Day</option>
            <option
                :value="row.toLowerCase()"
                v-for="row in rows"
                :selected="
                row.toLowerCase() ===
                (form.day === null ? null : form.day.toLowerCase())
              "
                :key="`row_option_${row.toLowerCase()}`"
                v-text="row"
            ></option>
          </select>
          <select class="form-control mt-2" v-model="form.data.icon">
            <option
                v-for="icon in icons"
                :value="icon.icon"
                :selected="form.icon === icon.icon"
                :key="`form_icon_${icon.icon}`"
            >
              <i :class="icon.icon"></i> {{ icon.title }}
            </option>
          </select>
          <input
              type="time"
              class="form-control mt-2"
              v-model="form.data.time.start"
          />
          <input
              type="time"
              class="form-control mt-2"
              v-model="form.data.time.finish"
          />
          <textarea
              class="form-control mt-2"
              rows="3"
              v-model="form.data.description"
          ></textarea>
          <div class="d-grid gap-2 no-print">
            <button
                v-if="!updateMode"
                class="btn btn-primary btn-block mt-2"
                type="button"
                @click="store"
            >
              Insert
            </button>
            <button
                v-if="updateMode"
                class="btn btn-success btn-block mt-2"
                type="button"
                @click="update"
            >
              Update
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import * as htmlToImage from 'html-to-image';

export default {
  name: "ScheduleTable",
  props: {
    rows: {
      type: Array,
      required: false,
      default: () => [
        "Monday",
        "Tuesday",
        "Wednesday",
        "Thursday",
        "Friday",
      ],
    },
  },
  data() {
    return {
      maxItems: 0,
      dataHeight: 0,
      viewerMode: false,
      updateMode: false,
      updatableIndex: null,
      errors: [],
      dataRows: {},
      form: {
        day: null,
        data: {
          icon: "bi bi-house",
          time: {
            start: null,
            finish: null,
          },
          description: null,
        },
      },
      icons: [
        {icon: "bi bi-alarm", title: "Wake Up"},
        {icon: "bi bi-moon-stars", title: "Sleep"},
        {icon: "bi bi-activity", title: "Running / Sport Activity"},
        {icon: "bi bi-person-workspace", title: "Working Shift Start"},
        {icon: "bi bi-person-video3", title: "Working Shift Off"},
        {icon: "bi bi-egg-fried", title: "Meal"},
        {icon: "bi bi-house", title: "General / None"},
      ],
    };
  },
  methods: {
    printSchedule: function () {
      htmlToImage.toJpeg(this.$refs["toPrint"], {quality: 1})
          .then(function (dataUrl) {
            let link = document.createElement('a');

            link.download = 'schedule.jpeg';
            link.href = dataUrl;
            link.click();
          });

    },
    deleteItem(day, index) {
      this.dataRows[day.toLowerCase()].splice(index, 1);
      this.storeData();
    },
    updateItem(day, index) {
      this.updateMode = true;
      this.form.day = day.toLowerCase();
      this.form.data = this.dataRows[this.form.day][index];
      this.updatableIndex = index;
    },
    update() {
      this.resetErrors();

      if (!this.validateForm(this.form)) {
        return;
      }

      this.dataRows[this.form.day][this.updatableIndex] = this.form.data;

      this.updateMode = false
      this.updatableIndex = null;

      this.resetForm();
      this.storeData();
    },
    store() {
      let form = this.form;

      this.resetErrors();

      if (!this.validateForm(form)) {
        return;
      }

      this.dataRows[this.form.day].push(form.data);
      this.resetForm();
      this.storeData();
    },
    validateForm(form) {
      if (!form.day) this.errors.push("Day is required");
      if (!form.data.icon) this.errors.push("Icon is required");
      if (!form.data.time.start) this.errors.push("Start Time is required");
      if (!form.data.time.finish) this.errors.push("Finish Time is required");
      if (!form.data.description) this.errors.push("Description is required");

      return this.errors.length === 0;
    },
    resetErrors() {
      this.errors = [];
    },
    resetForm() {
      this.form = {
        day: null,
        data: {
          icon: "bi bi-house",
          time: {
            start: null,
            finish: null,
          },
          description: null,
        },
      };
    },
    loadData() {
      let dataRows = localStorage.getItem("schedule");

      !dataRows
          ? localStorage.setItem("schedule", JSON.stringify(this.dataRows))
          : (this.dataRows = JSON.parse(dataRows));
    },
    storeData() {
      localStorage.setItem("schedule", JSON.stringify(this.dataRows));
    },
    clearData() {
      for (let row in this.rows) {
        this.dataRows[this.rows[row].toLowerCase()] = [];
      }
    },
    countMaxItems() {
      let count = 0;

      for (let row in this.rows) {
        let maxItems = this.dataRows[this.rows[row].toLowerCase()] || [];

        if (maxItems.length > count) {
          count = maxItems.length;
          console.log(count);
        }
      }

      this.maxItems = count;
    },
    getMaxContentHeight() {
      let element = this.$refs["maxHeight"];

      if (element) {
        console.log('maxDataHeight', element.clientHeight);
        this.dataHeight = element.clientHeight - Math.ceil(element.clientHeight * 0.1);
      }
    }
  },
  mounted() {
    this.clearData();
    this.loadData();
    this.countMaxItems();
  },
  updated() {
    this.getMaxContentHeight();
  },
  computed: {
    getMaxItems() {
      return new Array(this.maxItems);
    },
  },
};
</script>

<style scoped>
li {
  list-style: none !important;
}

li > i.bi {
  padding-right: 1em;
}

.data-div {
  overflow-x: auto !important;
  width: available !important;
}

p.error__list__item {
  line-height: 0.25em;
}

.day__item {
  background: rgba(10, 0, 0, 0.1);
  padding: .25em 0;
  border-radius: 2.5%;
}

.weekday__items {
  overflow: hidden;
  overflow-y: visible !important;
}

button.mode__changer:focus {
  outline: 0;
}
</style>
