<template>
    <view class="u-content">
        <rich-text :nodes="richNodes"></rich-text>
    </view>

</template>
<script>
    // 文本内容组件，优先兼容装修组件传入的 coreshopdata 结构
    export default {
        name: 'coreshopContent',
        props: {
            content: {
                type: [String, Object],
                default: ''
            },
            coreshopdata: {
                type: Object,
                default: function () {
                    return {};
                }
            }
        },
        computed: {
            richNodes() {
                if (typeof this.content === 'string' && this.content) {
                    return this.content;
                }

                if (this.coreshopdata && this.coreshopdata.parameters) {
                    const params = this.coreshopdata.parameters;
                    if (typeof params.content === 'string' && params.content) {
                        return params.content;
                    }
                    if (typeof params.value === 'string' && params.value) {
                        return params.value;
                    }
                }

                return '';
            }
        },
        created() { },
        methods: {
            preview(src, e) {
                // do something
            },
            navigate(href, e) {
                // do something
            }
        }
    }
</script>
<style lang="scss" scoped>
    .u-content { color: $u-content-color; font-size: 26rpx; line-height: 1.8; }
        .u-content p { color: $u-tips-color; }
</style>
