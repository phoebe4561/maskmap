<template>
  <div class="container-fluid ps-0 pe-0">
    <div class="sidenav" :class="{ toggle: isToggle === true }">
      <a href="#" class="closebtn" @click.prevent="isToggle = !isToggle">
        <i class="fa fa-angle-left fa-2"></i>
      </a>
      <div class="toolbox p-2">
        <div class="d-flex align-items-center justify-content-between">
          <h1 class="titleColor">口罩地圖</h1>
          <img src="./assets/mask.svg" alt="masklogo" width="100" height="100" />
        </div>
        <div class="today">
          <h2 class="today-week">星期{{ week }}</h2>
          <div class="today-info">
            <div class="today-date">{{ timestamp }}</div>
            <div v-if="today.getDay() % 7 === 0" class="today-description">
              身分證末碼為
              <span class="today-description-high-light"> 奇數或偶數 </span>
              可購買
            </div>
            <div v-else class="today-description">
              身分證末碼為
              <span class="today-description-high-light">
                {{ today.getDay() % 2 === 0 ? '0,2,4,6,8' : '1,3,5,7,9' }}
              </span>
              可購買
            </div>
          </div>
        </div>
        <div class="bg-lighter rounded p-2 my-2">
          <div class="form-group d-flex my-2">
            <label for="cityName" class="mr-2 col-form-label text-right">縣市: </label>
            <div class="flex-fill">
              <select id="cityName" class="form-control" v-model="select.city">
                <option value>--- 請選擇 ---</option>
                <option v-for="c in cityName" :key="c.CityName" :value="c.CityName">
                  {{ c.CityName }}
                </option>
              </select>
            </div>
          </div>
          <div class="form-group d-flex my-2">
            <label for="area" class="mr-2 col-form-label text-right">地區: </label>
            <div class="flex-fill">
              <select id="area" class="form-control" v-model="select.town">
                <option value>--- 請選擇 ---</option>
                <option v-for="town in getAreaName" :key="town.AreaName" :value="town.AreaName">
                  {{ town.AreaName }}
                </option>
              </select>
            </div>
          </div>
          <p class="mb-0 small text-muted text-right">請先選擇區域查看（淺橘色表示還有口罩）</p>
        </div>
        <ul class="list-group">
          <template v-for="item in filterData" :key="item.properties.id">
            <a
              @click="penTo(item)"
              :class="{
                highlight: item.properties.mask_adult || item.properties.mask_child,
              }"
              class="list-group-item bg-white text-secondary text-left"
            >
              <h3>{{ item.properties.name }}</h3>
              <p class="mb-0">成人口罩：{{ item.properties.mask_adult }}個 | 兒童口罩：{{ item.properties.mask_child }}個</p>
              <p class="mb-0">
                地址：
                <a target="_blank" title="Google Map" :href="`https://www.google.com.tw/maps/place/${item.properties.address}`">{{ item.properties.address }}</a>
              </p>
            </a>
          </template>
        </ul>
      </div>
    </div>
    <div class="container-fluid ps-0 pe-0">
      <div id="mymap"></div>
    </div>
  </div>
</template>

<script>
import cityName from './assets/cityName.json';

const { L } = window;
let osmMap = {};
const iconsConfig = {
  iconSize: [25, 41],
  iconAnchor: [12, 41],
  popupAnchor: [1, -34],
  shadowSize: [41, 41],
};
const icons = {
  green: new L.Icon({
    iconUrl: 'https://cdn.rawgit.com/pointhi/leaflet-color-markers/master/img/marker-icon-2x-green.png',
    ...iconsConfig,
  }),
  grey: new L.Icon({
    iconUrl: 'https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-grey.png',
    ...iconsConfig,
  }),
};

export default {
  data() {
    return {
      data: [],
      cityName,
      select: { city: '臺北市', town: '大安區' },
      nowData: [],
      today: new Date(),
      timestamp: '',
      weekList: ['日', '一', '二', '三', '四', '五', '六'],
      week: '',
      isToggle: false,
    };
  },
  methods: {
    getNow() {
      const date = `${this.today.getFullYear()}年${this.today.getMonth() + 1}月${this.today.getDate()}日`;
      this.timestamp = date;
    },
    getWeekDay() {
      const index = this.today.getDay();
      this.week = this.weekList[index];
    },
    penTo(item) {
      L.popup()
        .setLatLng([item.geometry.coordinates[1] + 0.0002, item.geometry.coordinates[0]])
        .setContent(
          `
        藥局名稱：${item.properties.name}
        <br>
        地址: <a href="https://www.google.com.tw/maps/place/${item.properties.address}" target="_blank">${item.properties.address}</a>
        <br>
        電話: ${item.properties.phone}
        <br>
        口罩數量：<strong>成人 ${item.properties.mask_adult ? `${item.properties.mask_adult} 個` : '未取得資料'}
        / 兒童 ${item.properties.mask_child ? `${item.properties.mask_child} 個` : '未取得資料'}</strong>
        <br>
        <small>最後更新時間: ${item.properties.updated}</small>`
        )
        .openOn(osmMap);
      osmMap.setView([item.geometry.coordinates[1], item.geometry.coordinates[0]], 18);
    },
    removeMarker() {
      // 把特定圖層的marker清空 直接將 addMarker 的 myGroup 刪除即可 這裡要注意 一開始 myGroup 會是空的 所以我們要加上判斷 myGroup 是否為空再執行
      if (this.myGroup) {
        this.myGroup.clearLayers();
      }
    },
    addMark() {
      const markers = new L.MarkerClusterGroup();
      this.data.forEach((item) => {
        const icon = item.properties.mask_adult || item.properties.mask_child ? icons.green : icons.grey;
        const layer = new L.Marker([item.geometry.coordinates[1], item.geometry.coordinates[0]], {
          icon,
        }).bindPopup(
          `
        藥局名稱：${item.properties.name}
        <br>
        地址: <a href="https://www.google.com.tw/maps/place/${item.properties.address}" target="_blank">${item.properties.address}</a>
        <br>
        電話: ${item.properties.phone}
        <br>
        口罩數量：<strong>成人 ${item.properties.mask_adult ? `${item.properties.mask_adult} 個` : '未取得資料'}
        / 兒童 ${item.properties.mask_child ? `${item.properties.mask_child} 個` : '未取得資料'}</strong>
        <br>
        <small>最後更新時間: ${item.properties.updated}</small>`
        );
        markers.addLayer(layer);
        osmMap.addLayer(markers);
      });
    },
  },
  computed: {
    getAreaName() {
      const arr = this.cityName.find((item) => {
        return item.CityName === this.select.city;
      });
      return arr.AreaList;
    },
    filterData() {
      const items = this.data.filter((item) => {
        return item.properties.county === this.select.city && item.properties.town === this.select.town;
      });
      // eslint-disable-next-line vue/no-side-effects-in-computed-properties
      this.nowData = items;
      this.removeMarker();
      this.addMark();
      if (items[0]) {
        this.penTo(items[0]);
      }
      return items;
    },
  },
  mounted() {
    osmMap = L.map('mymap', {
      center: [25.0286424, 121.5385066],
      zoom: 18,
    });
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution: '<a target="_blank" href="https://www.openstreetmap.org/">© OpenStreetMap 貢獻者</a>',
      maxZoom: 18,
    }).addTo(osmMap);

    const API = 'https://raw.githubusercontent.com/kiang/pharmacies/master/json/points.json';
    fetch(API)
      .then((response) => response.json())
      .then((res) => {
        this.data = res.features;
      });
  },
  created() {
    this.getNow();
    this.getWeekDay();
  },
};
</script>

<style lang="scss">
@import './assets/all';
#mymap {
  height: 100vh;
}
.highlight {
  background: $light !important;
}
.toolbox {
  background: $primary;
  height: 100vh;
  overflow-y: auto;
  overflow-x: hidden;
}

h3 {
  color: darken($primary, 20%);
}

.titleColor {
  color: #115555e5;
  font-weight: 400;
}

.toolbox::-webkit-scrollbar-track {
  box-shadow: inset 0 0 6px rgba(0, 0, 0, 0.3);
  background-color: #f5f5f5;
}

.toolbox::-webkit-scrollbar {
  width: 12px;
  background-color: #f5f5f5;
}

.toolbox::-webkit-scrollbar-thumb {
  border-radius: 10px;
  box-shadow: inset 0 0 6px rgba(0, 0, 0, 0.3);
  background-color: darken($primary, 30%);
}

.closebtn {
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px 6px;
  border-radius: 10px;
  background-color: darken($light, 10%);
  color: darken($light, 10%);
  position: absolute;
  top: 50%;
  right: -45px;
  transform: translateX(-50%) translateY(-50%);
  z-index: 900;
  transition: all 0.4s;
  &:hover {
    background-color: $primary;
    color: #fff;
    i {
      color: #fff;
    }
  }
  i {
    color: #73c0d8;
    font-size: 30px;
    transition: all 0.4s;
    transform: rotate(0deg);
  }
}

.sidenav {
  background-color: darken($light, 10%);
  box-shadow: 0px 3px 6px #000029;
  width: 350px;
  height: 100%;
  position: fixed;
  left: 0;
  z-index: 999;
  transition: all 0.5s ease;
  &.toggle {
    left: -350px;
    .closebtn {
      background-color: $primary;
      color: #fff;
      transition: all 0.4s;
      &:hover {
        background-color: darken($light, 10%);
      }
      i {
        color: #fff;
        transform: rotate(180deg);
      }
    }
  }
}
.leaflet-top {
  z-index: -10;
}
</style>
