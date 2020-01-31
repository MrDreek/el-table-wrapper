<script>
import Vue from 'vue'
import { getValueByPath, orderBy } from './util'
import { map } from 'lodash/core'
import debounce from 'lodash/debounce'
import groupBy from 'lodash/groupBy'

const defaultPagination = {
  currentPage: 1,
  pageSize: 10,
  layout: 'prev,pager,next'
}

const defaultFilterMultiple = true

const emptyFunction = () => {}

const emptyArrayFunction = () => []

export default {
  name: 'ElTableWrapper',
  props: {
    data: {
      type: Array,
      default: emptyArrayFunction
    },
    columns: {
      type: Array,
      default: emptyArrayFunction
    },
    showCustomHeader: {
      type: Boolean,
      default: false
    },
    columnDefault: {
      type: Object,
      default: emptyFunction
    },
    pagination: {
      type: Object,
      default: emptyFunction
    },
    defaultSort: {
      type: Object,
      default: emptyFunction
    },
    pinTitle: {
      type: Boolean,
      default: false
    },
    rowClassName: {
      type: Function,
      default: emptyFunction
    },
    maxHeight: {
      type: Number,
      default: 700
    }
  },
  data() {
    return {
      states: {
        filters: {},
        sortColumn: null,
        sortOrder: null,
        searches: {},
        pagination: this.getDefaultPagination()
      }
    }
  },
  computed: {
    localData() {
      const states = this.states
      const dataSource = this.data
      let data = dataSource || []
      // Sort locally
      data = data.slice(0)
      data = this.sortData(data)
      // filter
      if (states.filters) {
        Object.keys(states.filters).forEach((columnKey) => {
          const col = this.findColumn(columnKey)
          if (!col) {
            return
          }
          const values = states.filters[columnKey] || []
          if (values.length === 0) {
            return
          }
          let filterMethod = col.filterMethod
          if (col.searchable) {
            filterMethod = col.searchMethod || this.getDefaultSearchMethod(col)
          }
          data = filterMethod ? data.filter(record => {
            return values.some(v => filterMethod(v, record))
          }) : data
        })
      }
      return data
    },
    currentPageData() {
      let data = this.localData
      const pagination = this.states.pagination
      // If there is no pagination, show all by default
      if (!this.hasPagination()) {
        return data
      }

      const currentPage = this.getMaxCurrent(pagination.total || data.length)
      const pageSize = pagination.pageSize
      if (data.length > pageSize) {
        data = data.slice((currentPage - 1) * pageSize, currentPage * pageSize)
      }
      return data
    }
  },
  watch: {
    pagination: {
      handler: function(val) {
        if (val === false) {
          this.states.pagination = {}
          return
        }
        this.states.pagination = {
          ...defaultPagination,
          ...this.states.pagination,
          ...val,
          currentPage: val.currentPage || 1,
          pageSize: val.pageSize || 10
        }
      },
      deep: true
    }
  },
  mounted() {
    this.columns.map(columnAttr => {
      if (columnAttr.filters && columnAttr.filteredValue) {
        const values = columnAttr.filteredValue || []
        const key = this.getColumnKey(columnAttr)
        this.states.filters[key] = values
      }
    })

    if (this.defaulSort) {
      this.states.sortColumn = this.findColumnByProp(this.defaulSort.prop)
      this.states.sortOrder = this.defaulSort.order || 'ascending'
    }
  },
  methods: {
    onSortClick(event, { columnAttr, order, column }) {
      event.stopPropagation()

      let { sortColumn, sortOrder } = this.states
      // Only one column is allowed to be sorted at the same time, otherwise it will cause a logic problem of sort order
      const isSortColumn = this.isSortColumn(columnAttr)
      if (!isSortColumn) { // The current column is not sorted
        sortColumn = columnAttr
        sortOrder = order
      } else { // The current column is sorted
        if (sortOrder === order) {
          sortOrder = ''
          sortColumn = null
        } else { // Switch to sorted state
          sortOrder = order
        }
      }

      this.states.sortOrder = sortOrder
      this.states.sortColumn = sortColumn

      // TODO: Here rely on el-table to implement the bottom layer, to be abolished
      column.order = sortOrder

      this.$emit('sort-change', {
        column: column,
        columnAttr: sortColumn,
        prop: sortColumn ? sortColumn.prop : null,
        order: sortOrder
      })
    },
    onSearchClearClick(event, columnAttr) {
      event.stopPropagation()
      const key = this.getColumnKey(columnAttr)
      this.states.searches[key] = ''
      this.onSearchChange(columnAttr, '')
    },
    onSearchEnterPress(event, columnAttr) {
      event.stopPropagation()
      event.preventDefault()
      const key = this.getColumnKey(columnAttr)
      const searchValue = this.states.searches[key]
      this.onSearchChange(columnAttr, searchValue)
    },
    onSearchInput(columnAttr, value) {
      const key = this.getColumnKey(columnAttr)
      this.states.searches[key] = value
      if (columnAttr.searchOnInput) {
        this.onSearchChange(columnAttr, value)
      }
    },
    onSearchChange(columnAttr, value) {
      const key = this.getColumnKey(columnAttr)

      if (columnAttr.searchable && columnAttr.searchable === true) {
        this.states.filters = {
          ...this.states.filters,
          [key]: [value]
        }
      }

      if (this.hasPagination()) {
        const currentPage = 1
        this.onPageCurrentChange(currentPage)
      }

      this.$emit('search-change', {
        columnAttr: columnAttr,
        prop: columnAttr.prop,
        value: value
      })
    },
    onFilterChange(columnAttr, filteredValue) {
      let values = []
      if (filteredValue) {
        values = Array.isArray(filteredValue) ? filteredValue : [filteredValue]
      }
      const key = this.getColumnKey(columnAttr)

      this.onTableFilterChange({
        [key]: values
      })
    },
    onTableSortChange({ column, prop, order }) {
      const columnAttr = column ? this.findColumn(column.columnKey || column.property) : null
      this.states.sortColumn = columnAttr
      this.states.sortOrder = order
      this.$emit('sort-change', {
        column: column,
        columnAttr: columnAttr,
        prop,
        order
      })
    },
    onTableFilterChange(filters) {
      const nextFilters = {
        ...this.states.filters,
        ...filters
      }

      if (this.hasPagination()) {
        // Controlled current prop will not respond user interaction
        if (!(this.pagination && typeof this.pagination === 'object' &&
          'currentPage' in this.pagination)) {
          const currentPage = 1
          this.onPageCurrentChange(currentPage)
        }
      }

      this.states.filters = nextFilters
      this.$emit('filter-change', filters)
    },
    onPageSizeChange(size) {
      const pagination = this.states.pagination
      if (pagination.onSizeChange) {
        pagination.onSizeChange(size)
      }
      const nextPagination = {
        ...pagination,
        pageSize: size
      }
      this.states.pagination = nextPagination
      this.$emit('pagination-change', nextPagination)
    },
    onPageCurrentChange(currentPage) {
      const pagination = this.states.pagination
      if (pagination.onCurrentChange) {
        pagination.onCurrentChange(currentPage)
      }
      const nextPagination = {
        ...pagination,
        currentPage: currentPage || pagination.currentPage || 1
      }
      // Controlled current prop will not respond user interaction
      if (this.pagination && typeof this.pagination === 'object' &&
        'currentPage' in this.pagination) {
        nextPagination.currentPage = pagination.currentPage
      }
      this.states.pagination = nextPagination
      this.$emit('pagination-change', {
        ...nextPagination,
        currentPage: currentPage || pagination.currentPage || 1
      })
    },
    onHeaderTitleClick(e, { columnAttr }) {
      const filterable = columnAttr.filters
      if (filterable && this.showCustomHeader) {
        e.stopPropagation()
      }
    },
    onHeaderContentClick(e) {
      e.stopPropagation()
    },
    getDefaultPagination() {
      const pagination = this.pagination || {}
      return this.hasPagination() ? {
        ...defaultPagination,
        ...pagination
      } : {}
    },
    hasPagination() {
      return this.pagination !== false
    },
    sortData(data) {
      const states = this.states
      const { sortColumn, sortOrder } = states
      if (!sortColumn || typeof sortColumn.sortable === 'string') {
        return data
      }
      return orderBy(data, sortColumn.prop, sortOrder, sortColumn.sortMethod)
    },
    getDefaultSearchMethod(columnAttr) {
      const prop = columnAttr.prop
      return function(value, row) {
        const elementValue = prop && prop.indexOf('.') === -1
          ? row[prop] : getValueByPath(row, prop)
        const elementValueStr = elementValue.toString().toLowerCase()
        const valueStr = value.toString().toLowerCase()
        return elementValueStr.indexOf(valueStr) > -1
      }
    },
    getMaxCurrent(total) {
      const { currentPage, pageSize } = this.states.pagination
      if ((currentPage - 1) * pageSize >= total) {
        return Math.floor((total - 1) / pageSize) + 1
      }
      return currentPage
    },
    findColumn(myKey) {
      if (!myKey) {
        return null
      }
      let column
      this.columns.map(columnAttr => {
        if (this.getColumnKey(columnAttr) === myKey) {
          column = columnAttr
        }
      })
      return column
    },
    findColumnByProp(key) {
      if (!key) {
        return null
      }
      let column
      this.columns.map(columnAttr => {
        if (columnAttr.prop === key) {
          column = columnAttr
        }
      })
      return column
    },
    getColumnKey(columnAttr, index) {
      return columnAttr.columnKey || columnAttr.prop || index
    },
    isSortColumn(columnAttr) {
      const sortColumn = this.states.sortColumn
      if (!columnAttr || !sortColumn) {
        return false
      }
      return this.getColumnKey(sortColumn) === this.getColumnKey(columnAttr)
    },
    isFilterMultiple(columnAttr) {
      return Object.prototype.hasOwnProperty.call(columnAttr, 'filterMultiple')
        ? columnAttr.filterMultiple : defaultFilterMultiple
    },
    renderHeaderContentSearch(h, columnAttr, column) {
      const that = this
      const key = this.getColumnKey(columnAttr)
      if (!this.states.searches[key]) {
        Vue.set(this.states.searches, key, '')
      }
      const value = this.states.searches[key] || ''
      return (
        <el-input class="header-content-search" {...{
          props: {
            value: value
          },
          on: {
            input: debounce(value => {
              that.onSearchInput(columnAttr, value)
            }, 300)
          },
          nativeOn: {
            keyup: e => {
              if (e.keyCode === 13) { // Enter
                that.onSearchEnterPress(e, columnAttr)
              }
            }
          }
        }}>
          {
            value &&
            <i class="el-input__icon el-icon-close icon-search-delete" {...{
              slot: 'suffix',
              on: {
                click: e => that.onSearchClearClick(e, columnAttr)
              }
            }} />
          }
        </el-input>
      )
    },
    renderHeaderContentFilter(h, columnAttr, column) {
      const that = this
      const filters = columnAttr.filters
      const key = this.getColumnKey(columnAttr)
      if (!this.states.filters[key]) {
        Vue.set(this.states.filters, key, '')
      }
      const isMultiple = this.isFilterMultiple(columnAttr)
      let filterValue = this.states.filters[key]
      if (filterValue && !isMultiple) {
        filterValue = filterValue.length > 0 ? filterValue[0] : ''
      }
      return (
        <el-select class="header-content-filter" {...{
          props: {
            value: filterValue,
            placeholder: columnAttr.filterPlaceholder,
            multiple: isMultiple,
            clearable: true
          },
          on: {
            input: value => {
              that.states.filters[key] = value
              that.onFilterChange(columnAttr, value)
            },
            clear: () => {
              that.states.filters[key] = ''
              that.onFilterChange(columnAttr, '')
            }
          }
        }}>
          {
            filters && filters.map(filter => {
              return (
                <el-option {...{
                  props: {
                    label: filter.text,
                    value: filter.value
                  }
                }}>
                </el-option>
              )
            })
          }
        </el-select>
      )
    },
    renderHeaderContent(h, columnAttr, column, $index) {
      if (columnAttr.custom && columnAttr.custom.renderHeaderContent) {
        return columnAttr.custom.renderHeaderContent(h, { column, $index })
      }

      if (columnAttr.searchable) {
        return this.renderHeaderContentSearch(h, columnAttr, column)
      }

      const filterable = columnAttr.filters
      if (filterable) {
        return this.renderHeaderContentFilter(h, columnAttr, column)
      }

      return ''
    },
    getRenderHeaderFn(columnAttr) {
      const that = this
      /* eslint-disable */
      const headerAlign = columnAttr.headerAlign || columnAttr.align || ''
      /* eslint-enable */
      // TODO: sort classname should be here with table-header-title
      return function(h, { column, $index }) {
        return (
          <div class="table-header">
            <div class={['table-header-title', headerAlign]}
              on-click={e => that.onHeaderTitleClick(e, {
                columnAttr: columnAttr
              })}>
              <span>{columnAttr.label}</span>
              {
                columnAttr.sortable &&
                <span class="sort-caret-wrapper">
                  <span class="sort-icon-wrapper">
                    <i class="sort-icon el-icon-sort-up"
                      on-click={$event => that.onSortClick($event, {
                        column: column,
                        columnAttr: columnAttr,
                        order: 'ascending'
                      })}>
                    </i>
                  </span>
                  <span class="sort-icon-wrapper">
                    <i class="sort-icon el-icon-sort-down"
                      on-click={$event => that.onSortClick($event, {
                        column: column,
                        columnAttr: columnAttr,
                        order: 'descending'
                      })}>
                    </i>
                  </span>
                </span>
              }
            </div>
            <div class="table-header-content"
              on-click={e => that.onHeaderContentClick(e)}>
              {that.renderHeaderContent(h, columnAttr, column, $index)}
            </div>
          </div>
        )
      }
    },
    renderColumn(columnProps, tableProps) {
      const propsNoCustom = { ...columnProps }
      if (propsNoCustom.custom) {
        delete propsNoCustom.custom
      }

      if (tableProps.showCustomHeader) {
        propsNoCustom.renderHeader = this.getRenderHeaderFn(columnProps)
      }

      if (columnProps.searchable && columnProps.searchable === true) {
        propsNoCustom.filterMethod = null
        propsNoCustom.filters = null
        delete propsNoCustom.searchable
      }
      if (columnProps.filterMethod) {
        propsNoCustom.filterMethod = null
      }
      if (columnProps.sortable) {
        propsNoCustom.sortable = 'custom'
      }
      if (columnProps.scopedSlot) {
        delete propsNoCustom.scopedSlot
      }
      if (columnProps.type && columnProps.type === 'index') {
        propsNoCustom.className = propsNoCustom.className || ''
        propsNoCustom.className += ' ll-index'
      }

      return (
        <el-table-column {...{
          props: propsNoCustom
        }}>
          {
            columnProps && columnProps.scopedSlot
              ? this.$scopedSlots[columnProps.scopedSlot] : ''
          }
        </el-table-column>
      )
    },
    renderPagination() {
      if (!this.hasPagination) {
        return null
      }
      const { pagination } = this.states
      const total = pagination.total || this.localData.length
      if (!(total > 0)) {
        return null
      }
      return (
        <el-pagination class="ll-table-pagination" {...{
          props: {
            ...pagination,
            total: total,
            currentPage: this.getMaxCurrent(total)
          },
          on: {
            'size-change': this.onPageSizeChange,
            'current-change': this.onPageCurrentChange
          }
        }}>
        </el-pagination>
      )
    },
    renderGroupHeader(columns, tableOptions, label) {
      return (
        <el-table-column label={label}>
          {
            map(columns, column => this.renderColumn(column, tableOptions))
          }
        </el-table-column>
      )
    }
  },
  render() {
    const that = this
    const tableOptions = {
      showCustomHeader: this.showCustomHeader,
      data: this.currentPageData,
      defaultSort: this.defaultSort,
      rowClassName: this.rowClassName
    }
    const defaultColumnOptions = this.columnDefault || {}

    let props = {
      ...tableOptions,
      ...this.$attrs
    }

    if (this.pinTitle) {
      props = {
        ...props,
        ...{
          'max-height': this.maxHeight
        }
      }
    }

    const groupedColumns = groupBy(this.columns, 'groupName')
    return (
      <div class="ll-table-container">
        <el-table
          class={{ 'll-table': true, 'custom-header': this.showCustomHeader }}
          ref="ll-table" {...{
            props: props,
            on: {
              ...this.$listeners,
              'filter-change': that.onTableFilterChange,
              'sort-change': that.onTableSortChange
            },
            slot: 'append'
          }}>
          {
            this.$slots.append &&
            <div class="ll-table-body-append" slot='append'>
              {this.$slots.append}
            </div>
          }
          {
            map(groupedColumns, (columns, index) => {
              const columnOptions = {
                ...defaultColumnOptions,
                ...columns
              }
              if (index !== 'undefined') {
                return this.renderGroupHeader(columnOptions, tableOptions, columnOptions[0].groupNameLabel)
              }
              return map(columnOptions, column => this.renderColumn(column, tableOptions))
            })
          }
        </el-table>
        {this.renderPagination()}
      </div>
    )
  }
}
</script>

<style lang="scss">
$bordor-bottom-line-color: #e6ebf5;

.ll-table-pagination {
  margin: 16px 0;
  padding: 0;
  display: flex;
  justify-content: flex-end;
}

.ll-table.custom-header {
  th {
    padding: 0 0;
    vertical-align: top;

    div {
      padding: 0 0;
      line-height: 23px;
    }
  }

  th .cell {
    vertical-align: top;
    line-height: 14px;

    .caret-wrapper {
      display: none;
    }

    .el-table__column-filter-trigger {
      display: none;
    }
  }

  th.el-table-column--selection .cell {
    padding: 12px 10px;
    line-height: 23px;
    border-bottom: 1px solid $bordor-bottom-line-color;
  }

  th.ll-index .cell {
    padding: 12px 10px;
    line-height: 23px;
    border-bottom: 1px solid $bordor-bottom-line-color;
  }

  .table-header {
    width: 100%;
  }

  .table-header-title {
    box-sizing: border-box;
    padding: 12px 10px;
    line-height: 23px;
    border-bottom: 1px solid $bordor-bottom-line-color;

    display: flex;
    justify-content: flex-start;
    align-items: center;

    &.left {
      justify-content: flex-start;
    }

    &.right {
      justify-content: flex-end;
    }

    &.center {
      justify-content: center;
    }

    .sort-caret-wrapper {
      cursor: pointer;
      position: relative;
      display: inline-flex;
      align-items: center;
      width: 24px;
      height: 13px;
      line-height: 23px;
      overflow: initial;

      .sort-icon-wrapper {
        color: #878d99;
        width: 14px;
        overflow: hidden;
        font-size: 13px;
        line-height: 23px;
      }
    }
  }

  .ascending .sort-icon-wrapper .el-icon-sort-up {
    color: #409eff;
  }

  .descending .sort-icon-wrapper .el-icon-sort-down {
    color: #409eff;
  }

  .table-header-content {
    box-sizing: border-box;
    padding: 10px 10px;
    line-height: 23px;
    cursor: default;

    display: flex;
    justify-content: center;
    align-items: center;

    $height: 22px;

    .icon-search-delete {
      cursor: pointer;
      line-height: 23px;
    }

    .el-input {
      width: 100%;
      font-size: 12px;
    }

    .el-select {
      width: 100%;
      font-size: 12px;

      .el-input {
        input {
          height: 26px;
          max-height: 26px;
          overflow: hidden;
        }
      }

      .el-select__tags {
        background-color: transparent;
        height: $height;
        line-height: $height;
        max-height: $height;
        overflow: hidden;
      }

      .el-tag {
        height: $height;
        line-height: $height;
        margin: 0 0 0 6px;
      }
    }

    .el-input__inner {
      height: 26px;
    }
  }
}
</style>
