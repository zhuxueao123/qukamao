<template>
    <view class="store-page">
        <u-toast ref="uToast" /><u-no-network></u-no-network>
        <u-navbar title="附近商家"></u-navbar>

        <view class="map-wrap">
            <map class="map-body"
                 id="store-map"
                 :scale="12"
                 :show-location="true"
                 :show-compass="true"
                 :latitude="latitude"
                 :longitude="longitude"
                 :markers="markers">
            </map>
            <view class="map-tip">共找到 {{ renderList.length }} 个门店</view>
        </view>

        <view class="panel">
            <view class="search-line">
                <u-search placeholder="搜索门店名称或地址"
                          v-model="searchKey"
                          :show-action="true"
                          action-text="搜索"
                          :animation="false"
                          @clear="reloadList"
                          @search="reloadList"
                          @custom="reloadList"></u-search>
            </view>

            <view class="sort-line">
                <view class="sort-item" :class="{ active: sortType === 'distance' }" @tap="setSort('distance')">距离最近</view>
                <view class="sort-item" :class="{ active: sortType === 'name' }" @tap="setSort('name')">名称排序</view>
            </view>

            <scroll-view class="list-scroll" scroll-y>
                <view v-if="renderList.length" class="store-list">
                    <view class="store-card" v-for="item in renderList" :key="item.id" @tap="focusStore(item)">
                        <image class="store-logo" :src="item.logoImage" mode="aspectFill"></image>
                        <view class="store-content">
                            <view class="head-row">
                                <text class="store-name u-line-1">{{ item.storeName || '' }}</text>
                                <text class="store-distance">{{ item.distanceStr || '未知' }}</text>
                            </view>
                            <view class="store-address u-line-2">{{ item.address || '暂无地址' }}</view>
                            <view class="bottom-row">
                                <view class="phone" @tap.stop="doPhoneCall(item.mobile)">
                                    <u-icon name="phone" size="24" color="#666"></u-icon>
                                    <text>{{ item.mobile || '暂无电话' }}</text>
                                </view>
                                <view class="actions">
                                    <u-button size="mini" @click.stop="focusStore(item)">查看位置</u-button>
                                    <u-button type="primary" size="mini" @click.stop="goNavigation(item)">导航</u-button>
                                </view>
                            </view>
                        </view>
                    </view>
                </view>
                <view v-else class="coreshop-emptybox">
                    <u-empty :src="$globalConstVars.apiFilesUrl + '/static/images/empty/history.png'"
                             icon-size="280"
                             text="暂无门店列表"
                             mode="list"></u-empty>
                </view>
            </scroll-view>
        </view>
    </view>
</template>

<script>
    export default {
        data() {
            return {
                sourceList: [],
                renderList: [],
                markers: [],
                searchKey: '',
                sortType: 'distance',
                latitude: 0,
                longitude: 0,
                userLatitude: 0,
                userLongitude: 0,
                loading: false,
            };
        },
        onShow() {
            if (!this.sourceList.length) {
                this.initLocationAndLoad();
            }
        },
        onPullDownRefresh() {
            this.reloadList().finally(() => uni.stopPullDownRefresh());
        },
        methods: {
            initLocationAndLoad() {
                const failOver = () => {
                    this.userLatitude = 0;
                    this.userLongitude = 0;
                    this.latitude = 0;
                    this.longitude = 0;
                    this.fetchStoreList();
                };

                wx.getFuzzyLocation({
                    type: 'wgs84',
                    success: (res) => {
                        this.userLatitude = Number(res.latitude) || 0;
                        this.userLongitude = Number(res.longitude) || 0;
                        this.latitude = this.userLatitude;
                        this.longitude = this.userLongitude;
                        this.fetchStoreList();
                    },
                    fail: () => {
                        this.$u.toast('定位失败，已展示默认门店列表');
                        failOver();
                    }
                });
            },
            reloadList() {
                return this.fetchStoreList();
            },
            setSort(type) {
                if (this.sortType === type) {
                    return;
                }
                this.sortType = type;
                this.applyViewState();
            },
            fetchStoreList() {
                if (this.loading) {
                    return Promise.resolve();
                }
                this.loading = true;
                const payload = {
                    key: this.searchKey,
                    longitude: this.userLongitude,
                    latitude: this.userLatitude,
                    page: 1,
                    limit: 50,
                };

                return this.$u.api.storeList(payload).then((res) => {
                    if (!res.status) {
                        this.$u.toast(res.msg || '门店信息获取失败');
                        return;
                    }
                    this.sourceList = (res.data || []).map((item) => {
                        return {
                            ...item,
                            latitude: Number(item.latitude) || 0,
                            longitude: Number(item.longitude) || 0,
                            distance: Number(item.distance) || 0,
                            logoImage: item.logoImage || this.$globalConstVars.apiFilesUrl + '/static/images/logo.png',
                        };
                    });
                    this.applyViewState();
                }).finally(() => {
                    this.loading = false;
                });
            },
            applyViewState() {
                const list = [...this.sourceList];
                if (this.sortType === 'name') {
                    list.sort((a, b) => (a.storeName || '').localeCompare((b.storeName || ''), 'zh-CN'));
                } else {
                    list.sort((a, b) => a.distance - b.distance);
                }

                this.renderList = list;
                this.markers = list
                    .filter((item) => item.latitude && item.longitude)
                    .map((item) => {
                        return {
                            id: item.id,
                            latitude: item.latitude,
                            longitude: item.longitude,
                            iconPath: '/static/images/map/location.png',
                            width: 18,
                            height: 28,
                        };
                    });

                // 定位失败时，默认把地图中心移动到首个有效门店，避免地图停留在 0,0 导致灰屏感知。
                if (this.markers.length && (!this.latitude || !this.longitude)) {
                    this.latitude = this.markers[0].latitude;
                    this.longitude = this.markers[0].longitude;
                }
            },
            focusStore(item) {
                if (!item.latitude || !item.longitude) {
                    this.$u.toast('该门店暂无定位信息');
                    return;
                }
                this.latitude = item.latitude;
                this.longitude = item.longitude;
            },
            doPhoneCall(phone) {
                if (!phone || phone === '0') {
                    this.$u.toast('该门店暂无联系电话');
                    return;
                }
                uni.makePhoneCall({ phoneNumber: phone });
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
            }
        }
    };
</script>

<style lang="scss">
.store-page {
    min-height: 100vh;
    background: #f6f7fb;
}

.map-wrap {
    padding: 16rpx 16rpx 0;
    position: relative;
}

.map-body {
    width: 100%;
    height: 360rpx;
    border-radius: 16rpx;
}

.map-tip {
    position: absolute;
    left: 34rpx;
    bottom: 18rpx;
    background: rgba(0, 0, 0, 0.62);
    color: #fff;
    font-size: 22rpx;
    border-radius: 20rpx;
    padding: 8rpx 14rpx;
}

.panel {
    margin-top: 12rpx;
    background: #fff;
    border-radius: 20rpx 20rpx 0 0;
    padding: 16rpx;
}

.search-line {
    background: #f8f9fc;
    border-radius: 14rpx;
    padding: 8rpx 10rpx;
}

.sort-line {
    display: flex;
    gap: 24rpx;
    margin: 14rpx 8rpx;
}

.sort-item {
    font-size: 30rpx;
    color: #555;
    font-weight: 600;
}

.sort-item.active {
    color: #2f5fe9;
}

.list-scroll {
    height: calc(100vh - 44px - 360rpx - 178rpx);
}

.store-list {
    padding-bottom: 24rpx;
}

.store-card {
    background: #fff;
    border: 1px solid #eef1f6;
    border-radius: 16rpx;
    margin-bottom: 16rpx;
    padding: 16rpx;
    display: flex;
}

.store-logo {
    width: 96rpx;
    height: 96rpx;
    border-radius: 12rpx;
    background: #f3f3f3;
    flex-shrink: 0;
}

.store-content {
    flex: 1;
    margin-left: 14rpx;
    min-width: 0;
}

.head-row {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 12rpx;
}

.store-name {
    flex: 1;
    min-width: 0;
    font-size: 30rpx;
    color: #1f2433;
    font-weight: 700;
}

.store-distance {
    font-size: 22rpx;
    color: #2d8cf0;
    background: #e9f5ff;
    border-radius: 10rpx;
    padding: 4rpx 10rpx;
}

.store-address {
    margin-top: 10rpx;
    font-size: 24rpx;
    color: #666;
    line-height: 1.45;
}

.bottom-row {
    margin-top: 12rpx;
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 10rpx;
}

.phone {
    display: flex;
    align-items: center;
    font-size: 22rpx;
    color: #666;
    flex: 1;
    min-width: 0;
}

.phone text {
    margin-left: 8rpx;
}

.actions {
    display: flex;
    gap: 10rpx;
}
</style>
