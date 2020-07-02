<template>
    <div class="resource-layout">
        <host-list-options></host-list-options>
        <bk-table class="hosts-table"
            v-bkloading="{ isLoading: $loading(Object.values(request)) }"
            :data="table.list"
            :pagination="table.pagination"
            :row-style="{ cursor: 'pointer' }"
            :max-height="$APP.height - 230"
            @selection-change="handleSelectionChange"
            @row-click="handleRowClick"
            @sort-change="handleSortChange"
            @page-change="handlePageChange"
            @page-limit-change="handleSizeChange">
            <bk-table-column type="selection" width="60" align="center" fixed class-name="bk-table-selection"></bk-table-column>
            <bk-table-column v-for="property in table.header"
                :key="property.bk_property_id"
                :label="$tools.getHeaderPropertyName(property)"
                :sortable="isPropertySortable(property) ? 'custom' : false"
                :prop="property.bk_property_id"
                :fixed="property.bk_property_id === 'bk_host_innerip'"
                :class-name="property.bk_property_id === 'bk_host_innerip' ? 'is-highlight' : ''">
                <template slot-scope="{ row }">
                    {{ row | hostValueFilter(property.bk_obj_id, property.bk_property_id) | formatter(property)}}
                </template>
            </bk-table-column>
            <cmdb-table-empty slot="empty" :stuff="table.stuff"></cmdb-table-empty>
        </bk-table>
    </div>
</template>

<script>
    import { mapGetters } from 'vuex'
    import hostListOptions from './host-options.vue'
    import hostValueFilter from '@/filters/host'
    import {
        MENU_RESOURCE_HOST_DETAILS,
        MENU_RESOURCE_BUSINESS_HOST_DETAILS
    } from '@/dictionary/menu-symbol'
    import { getIPPayload, injectFields, injectAsset } from '@/utils/host'
    import RouterQuery from '@/router/query'
    export default {
        components: {
            hostListOptions
        },
        filters: {
            hostValueFilter
        },
        data () {
            return {
                properties: {
                    biz: [],
                    host: [],
                    set: [],
                    module: []
                },
                propertyGroups: [],
                directory: null,
                scope: 1,
                table: {
                    checked: [],
                    header: Array(8).fill({}),
                    list: [],
                    pagination: {
                        current: 1,
                        count: 0,
                        ...this.$tools.getDefaultPaginationConfig()
                    },
                    sort: 'bk_host_id',
                    exportUrl: `${window.API_HOST}hosts/export`,
                    stuff: {
                        type: 'default',
                        payload: {
                            emptyText: this.$t('bk.table.emptyText')
                        }
                    }
                },
                columnsConfig: {
                    selected: []
                },
                columnsConfigDisabledColumns: ['bk_host_innerip', 'bk_cloud_id', 'bk_biz_name', 'bk_module_name'],
                request: {
                    property: Symbol('property'),
                    propertyGroup: Symbol('propertyGroup'),
                    list: Symbol('list')
                }
            }
        },
        computed: {
            ...mapGetters(['userName']),
            ...mapGetters('userCustom', ['usercustom']),
            ...mapGetters('hosts', ['condition']),
            ...mapGetters('resourceHost', ['activeDirectory']),
            customColumns () {
                return this.usercustom[this.$route.meta.customInstanceColumn] || []
            },
            columnsConfigProperties () {
                const setProperties = this.properties.set.filter(property => ['bk_set_name'].includes(property['bk_property_id']))
                const moduleProperties = this.properties.module.filter(property => ['bk_module_name'].includes(property['bk_property_id']))
                const businessProperties = this.properties.biz.filter(property => ['bk_biz_name'].includes(property['bk_property_id']))
                const hostProperties = this.properties.host
                return [...setProperties, ...moduleProperties, ...businessProperties, ...hostProperties]
            }
        },
        watch: {
            customColumns () {
                this.setTableHeader()
            },
            columnsConfigProperties () {
                this.setTableHeader()
            },
            scope () {
                this.setModuleNamePropertyState()
            }
        },
        async created () {
            try {
                await Promise.all([
                    this.getProperties(),
                    this.getHostPropertyGroups()
                ])
                this.setModuleNamePropertyState()
                this.unwatch = RouterQuery.watch('*', ({
                    scope = 1,
                    page = 1,
                    sort = 'bk_host_id',
                    limit = this.table.pagination.limit,
                    directory = null
                }) => {
                    this.table.pagination.current = parseInt(page)
                    this.table.pagination.limit = parseInt(limit)
                    this.table.sort = sort
                    this.directory = parseInt(directory) || null
                    this.scope = isNaN(scope) ? 'all' : parseInt(scope)
                    this.getHostList()
                }, { immediate: true, throttle: 100 })
            } catch (error) {
                console.error(error)
            }
        },
        beforeDestroy () {
            this.unwatch()
        },
        methods: {
            async getProperties () {
                try {
                    const propertyMap = await this.$store.dispatch('objectModelProperty/batchSearchObjectAttribute', {
                        params: {
                            bk_obj_id: { '$in': Object.keys(this.properties) },
                            bk_supplier_account: this.supplierAccount
                        },
                        config: {
                            requestId: this.request.property
                        }
                    })
                    this.properties = propertyMap
                } catch (error) {
                    console.error(error)
                }
            },
            async getHostPropertyGroups () {
                try {
                    this.propertyGroups = await this.$store.dispatch('objectModelFieldGroup/searchGroup', {
                        objId: 'host',
                        params: {},
                        config: {
                            requestId: this.request.propertyGroup
                        }
                    })
                } catch (error) {
                    console.error(error)
                }
            },
            setModuleNamePropertyState () {
                const property = this.properties.module.find(property => property.bk_property_id === 'bk_module_name')
                if (property) {
                    const normalName = this.$t('模块名')
                    const directoryName = this.$t('目录名')
                    const scopeModuleName = {
                        0: normalName,
                        1: directoryName,
                        all: `${directoryName}/${normalName}`
                    }
                    property.bk_property_name = scopeModuleName[this.scope]
                }
            },
            setTableHeader () {
                const customColumns = this.customColumns.length ? this.customColumns : this.globalCustomColumns
                this.table.header = this.$tools.getHeaderProperties(this.columnsConfigProperties, customColumns, this.columnsConfigDisabledColumns)
                this.columnsConfig.selected = this.table.header.map(property => property['bk_property_id'])
            },
            isPropertySortable (property) {
                return property.bk_obj_id === 'host' && !['foreignkey'].includes(property.bk_property_type)
            },
            async getHostList (event) {
                try {
                    const { count, info } = await this.$store.dispatch('hostSearch/searchHost', {
                        params: this.getParams(),
                        config: {
                            requestId: this.request.list
                        }
                    })
                    this.table.pagination.count = count
                    this.table.list = info
                    this.table.stuff.type = event ? 'search' : 'default'
                } catch (error) {
                    this.table.pagination.count = 0
                    this.table.checked = []
                    this.table.list = []
                    console.error(error)
                }
            },
            getParams () {
                const params = {
                    bk_biz_id: -1,
                    ip: getIPPayload(),
                    page: {
                        ...this.$tools.getPageParams(this.table.pagination),
                        sort: this.table.sort
                    },
                    condition: this.$tools.clone(this.condition)
                }
                injectFields(params, this.table.header)
                injectAsset(params, RouterQuery.get('bk_asset_id'))
                this.injectScope(params)
                this.injectDirectory(params)
                return params
            },
            injectScope (params) {
                const biz = params.condition.find(condition => condition.bk_obj_id === 'biz')
                if (this.scope === 'all') {
                    biz.condition = biz.condition.filter(condition => condition.field !== 'default')
                } else {
                    const newMeta = {
                        field: 'default',
                        operator: '$eq',
                        value: this.scope
                    }
                    const existMeta = biz.condition.find(({ field, operator }) => field === newMeta.field && operator === newMeta.operator)
                    if (existMeta) {
                        existMeta.value = newMeta.value
                    } else {
                        biz.condition.push(newMeta)
                    }
                }
                return params
            },
            injectDirectory (params) {
                if (!this.directory) {
                    return false
                }
                const moduleCondition = params.condition.find(condition => condition.bk_obj_id === 'module')
                const directoryMeta = {
                    field: 'bk_module_id',
                    operator: '$eq',
                    value: this.directory
                }
                const existMeta = moduleCondition.condition.find(({ field, operator }) => field === directoryMeta.field && operator === directoryMeta.operator)
                if (existMeta) {
                    existMeta.value = directoryMeta.value
                } else {
                    moduleCondition.condition.push(directoryMeta)
                }
            },
            handleSelectionChange (selection) {
                this.table.checked = selection.map(item => item.host.bk_host_id)
            },
            handleRowClick (item) {
                const business = item.biz[0]
                if (business.default) {
                    this.$router.push({
                        name: MENU_RESOURCE_HOST_DETAILS,
                        params: {
                            id: item.host.bk_host_id
                        },
                        query: {
                            from: 'resource'
                        }
                    })
                } else {
                    this.$router.push({
                        name: MENU_RESOURCE_BUSINESS_HOST_DETAILS,
                        params: {
                            business: business.bk_biz_id,
                            id: item.host.bk_host_id
                        },
                        query: {
                            from: 'resource'
                        }
                    })
                }
            },
            handlePageChange (current) {
                RouterQuery.set({
                    page: current,
                    _t: Date.now()
                })
            },
            handleSizeChange (limit) {
                RouterQuery.set({
                    limit: limit,
                    page: 1,
                    _t: Date.now()
                })
            },
            handleSortChange (sort) {
                RouterQuery.set({
                    sort: this.$tools.getSort(sort),
                    _t: Date.now()
                })
            }
        }
    }
</script>

<style lang="scss" scoped>
</style>