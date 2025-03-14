<template>
    <div class="history-layout">
        <div class="history-options clearfix">
            <label class="fl">{{$t('Common["已删除历史"]')}}</label>
            <bk-button class="fr" type="primary" @click="back">{{$t('Common["返回"]')}}</bk-button>
            <bk-date-range class="history-date-range fr"
                :ranges="ranges"
                :range-separator="'-'"
                :quick-select="true"
                :start-date="startDate"
                :end-date="endDate"
                position="bottom-left"
                @change="setFilterTime">
            </bk-date-range>
        </div>
        <cmdb-table class="history-table"
            row-cursor="default"
            :sortable="false"
            :loading="$loading()"
            :pagination.sync="pagination"
            :list="list"
            :header="header"
            :wrapper-minus-height="157"
            @handlePageChange="handlePageChange"
            @handleSizeChange="handleSizeChange">
        </cmdb-table>
    </div>
</template>

<script>
    import { mapGetters, mapActions } from 'vuex'
    import moment from 'moment'
    export default {
        data () {
            const startDate = this.$tools.formatTime(moment().subtract(1, 'month'), 'YYYY-MM-DD')
            const endDate = this.$tools.formatTime(moment(), 'YYYY-MM-DD')
            return {
                properties: [],
                header: [],
                list: [],
                pagination: {
                    current: 1,
                    size: 10,
                    count: 0
                },
                opTime: [],
                startDate,
                endDate,
                opTimeResolver: null
            }
        },
        computed: {
            ...mapGetters(['supplierAccount', 'userName', 'isAdminView']),
            ...mapGetters('userCustom', ['usercustom']),
            ...mapGetters('objectBiz', ['bizId']),
            customColumns () {
                const customKeyMap = {
                    [this.objId]: `${this.userName}_${this.objId}_${this.isAdminView ? 'adminView' : this.bizId}_table_columns`,
                    'host': `${this.userName}_$resource_${this.isAdminView ? 'adminView' : this.bizId}_table_columns`
                }
                return this.usercustom[customKeyMap[this.objId]] || []
            },
            objId () {
                return this.$route.params.objId
            },
            ranges () {
                const language = this.$i18n.locale
                if (language === 'en') {
                    return {
                        'Yesterday': [moment().subtract(1, 'days'), moment()],
                        'Last Week': [moment().subtract(7, 'days'), moment()],
                        'Last Month': [moment().subtract(1, 'month'), moment()],
                        'Last Three Month': [moment().subtract(3, 'month'), moment()]
                    }
                }
                return {
                    昨天: [moment().subtract(1, 'days'), moment()],
                    最近一周: [moment().subtract(7, 'days'), moment()],
                    最近一个月: [moment().subtract(1, 'month'), moment()],
                    最近三个月: [moment().subtract(3, 'month'), moment()]
                }
            }
        },
        watch: {
            opTime (opTime) {
                if (this.opTimeResolver) {
                    this.opTimeResolver()
                } else {
                    this.handlePageChange(1)
                }
            }
        },
        async created () {
            try {
                await this.setTimeResolver()
                this.properties = await this.searchObjectAttribute({
                    params: this.$injectMetadata({
                        bk_obj_id: this.objId,
                        bk_supplier_account: this.supplierAccount
                    }),
                    config: {
                        requestId: `post_searchObjectAttribute_${this.objId}`,
                        fromCache: false
                    }
                })
                await this.setTableHeader()
                this.getTableData()
            } catch (e) {
                // ignore
            }
        },
        methods: {
            ...mapActions('objectModelProperty', ['searchObjectAttribute']),
            ...mapActions('operationAudit', ['getOperationLog']),
            back () {
                this.$router.go(-1)
            },
            setTimeResolver () {
                return new Promise((resolve, reject) => {
                    this.opTimeResolver = () => {
                        this.opTimeResolver = null
                        resolve()
                    }
                })
            },
            setTableHeader () {
                const idMap = {
                    'host': 'bk_host_id',
                    'set': 'bk_set_id',
                    'module': 'bk_module_id',
                    'biz': 'bk_biz_id',
                    'plat': 'bk_plat_id'
                }
                const fixedPropertyMap = {
                    'host': ['bk_host_innerip', 'bk_cloud_id'],
                    'set': ['bk_set_name'],
                    'module': ['bk_module_name'],
                    'biz': ['bk_biz_name'],
                    'plat': ['bk_plat_name']
                }
                const headerProperties = this.$tools.getHeaderProperties(this.properties, this.customColumns, fixedPropertyMap[this.objId] || ['bk_inst_name'])
                this.header = [{
                    id: idMap[this.objId] || 'bk_inst_id',
                    name: 'ID'
                }].concat(headerProperties.map(property => {
                    return {
                        id: property['bk_property_id'],
                        name: property['bk_property_name']
                    }
                })).concat([{
                    id: 'op_time',
                    width: 180,
                    name: this.$t('Common["更新时间"]')
                }])
                return Promise.resolve(this.header)
            },
            setFilterTime (oldVal, newVal) {
                this.opTime = newVal.split(' - ').map((date, index) => {
                    return index === 0 ? (date + ' 00:00:00') : (date + ' 23:59:59')
                })
            },
            getTableData () {
                this.getOperationLog({
                    params: this.getSearchParams(),
                    config: {
                        cancelPrevious: true,
                        requestId: `search${this.objId}OperationLog`
                    }
                }).then(log => {
                    try {
                        this.pagination.count = log.count
                        const list = log.info.map(data => {
                            return {
                                ...(data.content['cur_data'] ? data.content['cur_data'] : data.content['pre_data']),
                                'op_time': this.$tools.formatTime(data['op_time'])
                            }
                        })
                        this.list = this.$tools.flatternList(this.properties, list)
                    } catch (e) {
                        this.list = []
                        this.$error(e.message)
                    }
                })
            },
            getSearchParams () {
                return {
                    condition: {
                        'op_type': 3,
                        'op_time': this.opTime,
                        'op_target': this.objId
                    },
                    start: (this.pagination.current - 1) * this.pagination.size,
                    limit: this.pagination.size,
                    sort: '-op_time'
                }
            },
            handleSizeChange (size) {
                this.pagination.size = size
                this.handlePageChange(1)
            },
            handlePageChange (current) {
                this.pagination.current = current
                this.getTableData()
            }
        }
    }
</script>

<style lang="scss" scoped>
    .history-layout{
        padding: 20px;
    }
    .history-options{
        height: 36px;
        line-height: 36px;
        font-size: 14px;
    }
    .history-table{
        margin-top: 20px;
    }
</style>

<style lang="scss">
    .history-date-range{
        .range-action{
            display: none;
        }
    }
</style>
