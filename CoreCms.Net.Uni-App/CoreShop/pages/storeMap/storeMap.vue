<template>
    <view class="store-page">
        <u-toast ref="uToast" /><u-no-network></u-no-network>
        <u-navbar title="附近商家"></u-navbar>

        <view class="map-wrap">
            <map class="map-body"
                 id="map"
                 :scale="12"
                 :show-location="true"
                 :show-compass="true"
                 :enable-overlooking="true"
                 :latitude="latitude"
                 :longitude="longitude"
                 :markers="covers">
            </map>
            <view class="found-bar">📍 共找到 {{ storeList.length }} 个门店</view>
        </view>

        <view class="panel">
            <view class="search-box">
                <u-search placeholder="搜索门店名称或地址"
                          v-model="searchKey"
                          :show-action="true"
                          action-text="搜索"
                          :animation="false"
                          @clear="onClearSearch"
                          @search="onSearch"
                          @custom="onSearch"></u-search>
            </view>

            <view class="sort-row">
                <view class="sort-item"
                      :class="{ active: sortType === 'distance' }"
                      @tap="changeSort('distance')">距离最近</view>
                <view class="sort-item"
                      :class="{ active: sortType === 'name' }"
                      @tap="changeSort('name')">名称排序</view>
            </view>

            <scroll-view class="store-list" scroll-y>
                <view v-if="storeList.length > 0" class="card-list">
                    <view class="store-card" v-for="item in storeList" :key="item.id" @tap="focusStore(item)">
                        <view class="card-top">
                            <image class="store-logo" :src="item.logoImage" mode="aspectFill"></image>
                            <view class="store-main">
                                <view class="title-row">
                                    <text class="store-name u-line-1">{{ item.storeName || '' }}</text>
                                    <text class="distance">{{ item.distanceStr || '未知' }}</text>
                                </view>
                                <view class="status-row">
                                    <text class="dot"></text>
                                    <text class="status">可自提</text>
                                </view>
                                <view class="addr u-line-2">{{ item.address || '暂无地址' }}</view>
                            </view>
                        </view>

                        <view class="card-bottom">
                            <view class="phone" @tap.stop="doPhoneCall(item.mobile)">
                                <u-icon name="phone" size="26" color="#5e5e5e"></u-icon>
                                <text>{{ item.mobile || '暂无电话' }}</text>
                            </view>
                            <view class="btns">
                                <u-button size="mini" shape="circle" @click.stop="focusStore(item)">查看位置</u-button>
                                <u-button size="mini" shape="circle" type="primary" @click.stop="goNavigation(item)">导航</u-button>
                            </view>
                        </view>
                    </view>
                </view>

                <view class="coreshop-emptybox" v-else>
                    <u-empty :src="$globalConstVars.apiFilesUrl + '/static/images/empty/history.png'"
                             icon-size="300"
                             text="暂无门店列表"
                             mode="list"></u-empty>
                </view>
            </scroll-view>
        </view>

        <coreshop-login-modal></coreshop-login-modal>
    </view>
</template>

<script>
    export default {
        data() {
            return {
                sourceList: [],
                storeList: [],
                searchKey: '',
                sortType: 'distance',
                longitude: 0,
                latitude: 0,
                userLongitude: 0,
                userLatitude: 0,
                covers: [],
                page: 1,
                limit: 30,
                isLoading: false,
            };
        },
        onLoad() {
            this.getMyLocation();
        },
        onPullDownRefresh() {
            this.refreshStores();
        },
        methods: {
            refreshStores() {
                this.page = 1;
                this.getStoreList().finally(() => {
                    uni.stopPullDownRefresh();
                });
            },
            onSearch() {
                this.page = 1;
                this.getStoreList();
            },
            onClearSearch() {
                this.searchKey = '';
                this.page = 1;
                this.getStoreList();
            },
            changeSort(type) {
                if (this.sortType === type) {
                    return;
                }
                this.sortType = type;
                this.applyListState();
            },
            // 获取定位，失败时使用0坐标继续查询列表
            getMyLocation() {
                let _this = this;
                wx.getFuzzyLocation({
                    type: 'wgs84',
                    success(res) {
                        _this.userLongitude = Number(res.longitude) || 0;
                        _this.userLatitude = Number(res.latitude) || 0;
                        _this.longitude = _this.userLongitude;
                        _this.latitude = _this.userLatitude;
                        _this.getStoreList();
                    },
                    fail() {
                        _this.userLongitude = 0;
                        _this.userLatitude = 0;
                        _this.longitude = 0;
                        _this.latitude = 0;
                        _this.$u.toast('获取定位失败，已按默认位置展示门店');
                        _this.getStoreList();
                    }
                });
            },
            // 获取门店列表信息
            getStoreList() {
                if (this.isLoading) {
                    return Promise.resolve();
                }

                this.isLoading = true;
                const data = {
                    key: this.searchKey,
                    longitude: this.userLongitude,
                    latitude: this.userLatitude,
                    page: this.page,
                    limit: this.limit,
                };

                return this.$u.api.storeList(data).then(res => {
                    if (!res.status) {
                        this.$u.toast('门店信息获取失败');
                        return;
                    }

                    this.sourceList = (res.data || []).map(item => {
                        const latitude = Number(item.latitude) || 0;
                        const longitude = Number(item.longitude) || 0;
                        return {
                            ...item,
                            latitude,
                            longitude,
                            distance: Number(item.distance) || 0,
                            logoImage: item.logoImage || this.$globalConstVars.apiFilesUrl + '/static/images/logo.png',
                        };
                    });

                    this.applyListState();
                }).finally(() => {
                    this.isLoading = false;
                });
            },
            applyListState() {
                const list = [...this.sourceList];

                if (this.sortType === 'name') {
                    list.sort((a, b) => (a.storeName || '').localeCompare((b.storeName || ''), 'zh-CN'));
                } else {
                    list.sort((a, b) => a.distance - b.distance);
                }

                this.storeList = list;
                this.buildMarkers();
            },
            buildMarkers() {
                const markers = this.storeList
                    .filter(item => item.latitude && item.longitude)
                    .map(item => {
                        return {
                            id: item.id,
                            latitude: item.latitude,
                            longitude: item.longitude,
                            iconPath: '/static/images/map/location.png',
                            width: 20,
                            height: 30,
                        };
                    });

                this.covers = markers;
            },
            doPhoneCall(phone) {
                if (!phone || phone === '0') {
                    this.$u.toast('该门店暂无联系电话');
                    return;
                }
                uni.makePhoneCall({ phoneNumber: phone });
            },
            focusStore(item) {
                if (!item.latitude || !item.longitude) {
                    this.$u.toast('该门店暂无定位信息');
                    return;
                }
                this.latitude = item.latitude;
                this.longitude = item.longitude;
            },
            goNavigation(item) {
                if (!item.latitude || !item.longitude) {
                    this.$u.toast('该门店暂无定位信息');
                    return;
                }
                uni.openLocation({
                    latitude: item.latitude,
                    longitude: item.longitude,
                    name: item.storeName || '门店位置',
                    address: item.address || '',
                    scale: 16,
                });
            },
        }
    };
</script>

<style scoped lang="scss">
    @import "storeMap.scss";
</style>
