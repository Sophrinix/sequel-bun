<template>
  <div class="section no-padding is-fullheight">

    <!-- Context menu -->
    <context-menu ref="ctxMenu" @ctx-open="onCtxOpen">
      <div class="dropdown is-active">
        <div class="dropdown-menu" role="menu">
          <div class="dropdown-content">
            <a href="#" class="dropdown-item">
              Truncate
            </a>
            <hr class="dropdown-divider">
            <a href="#" class="dropdown-item has-text-danger" @click="deleteConn()">
              Drop table
            </a>
          </div>
        </div>
      </div>
    </context-menu>

    <div class="columns is-fullheight">
      <aside class="column is-one-quarter menu padding-left">

        <!-- Show databases if not selected -->
        <div v-if="$parent.db === null" class="is-fullheight">
          <p class="menu-label has-text-weight-bold">Databases</p>
          <ul class="menu-list overflow" v-if="databases.length > 0">
            <li v-for="db in databases" @contextmenu.prevent="$refs.ctxMenu.open($event, {db: db})" :key="db">
              <a @click="use(db)">
                <img src="static/database-small.png" :alt="'Database ' + db" />
                <span>{{ db }}</span>
              </a>
            </li>
          </ul>
          <small class="has-text-grey" v-else>No database found.</small>
        </div>

        <!-- Otherwise, show tables -->
        <div class="is-fullheight" v-else>
          <!-- Change database -->
          <p class="menu-label has-text-weight-bold">Databases</p>
          <div class="select">
            <select v-model="$parent.db" @change="use($parent.db)">
              <option value="-1" disabled>Choose database</option>
              <option v-for="db in databases" :value="db" :key="db">{{ db }}</option>
            </select>
          </div>

          <!-- Tables list -->
          <p class="menu-label has-text-weight-bold">
            Tables
            <span v-if="table">-</span>
            <span v-if="table" class="weight-500 truncate">{{ table }}</span>
          </p>
          <div v-if="tables.length > 0" id="tables-container">
            <input type="text" class="input" v-model="filterTable" placeholder="Search table" />

            <ul class="menu-list overflow margin-top-5">
              <li v-for="t in searchTable(tables)" @contextmenu.prevent="$refs.ctxMenu.open($event, {table: t})" :key="t">
                <a @click="structure(t)" :class="{'is-active': t == table}">
                  <img src="static/table-small.png" :alt="'Table ' + t" />
                  <span>{{ t }}</span>
                </a>
              </li>
            </ul>
          </div>
          <small class="has-text-grey" v-else>No table found.</small>
        </div>
      </aside>

      <main class="column no-padding">
        <router-view ref="connected"></router-view>
      </main>
    </div><!-- /.columns -->
  </div><!-- /.section -->
</template>

<script>
  export default {
    name: 'connected',
    data: () => ({
      ctxData: {},
      conn: null, // Connection instance
      filterTable: '', // Search tables
      // Current table
      table: null,
      columns: [],
      primary_key: null,
      foreign_keys: {},
      // Cache
      databases: [], // List current conn. dbs
      tables: [], // List current conn. tables
      tables_columns: {}, // All tables columns
      tables_primary_keys: {},
      tables_foreign_keys: {},
      // Column types
      common_column_types: [
        {
          name: 'INT',
          default: 11,
          group: false,
          children: []
        },
        {
          name: 'VARCHAR',
          default: 255,
          group: false,
          children: []
        },
        {
          name: 'TEXT',
          default: null,
          group: false,
          children: []
        },
        {
          name: 'DATETIME',
          default: null,
          group: false,
          children: []
        }
      ],
      column_types: [
        {
          name: 'Numeric',
          default: null,
          group: true,
          children: [
            {
              name: 'TINYINT',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'SMALLINT',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'MEDIUMINT',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'INT',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'BIGINT',
              default: null,
              group: false,
              children: []
            },
            {
              name: '-',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'DECIMAL',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'FLOAT',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'DOUBLE',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'REAL',
              default: null,
              group: false,
              children: []
            },
            {
              name: '-',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'BIT',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'BOOLEAN',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'SERIAL',
              default: null,
              group: false,
              children: []
            }
          ]
        },
        {
          name: 'Date and time',
          default: null,
          group: true,
          children: [
            {
              name: 'DATE',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'DATETIME',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'TIMESTAMP',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'TIME',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'YEAR',
              default: null,
              group: false,
              children: []
            }
          ]
        },
        {
          name: 'String',
          default: null,
          group: true,
          children: [
            {
              name: 'CHAR',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'VARCHAR',
              default: 255,
              group: false,
              children: []
            },
            {
              name: '-',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'TINYTEXT',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'TEXT',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'MEDIUMTEXT',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'LONGTEXT',
              default: null,
              group: false,
              children: []
            },
            {
              name: '-',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'BINARY',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'VARBINARY',
              default: null,
              group: false,
              children: []
            },
            {
              name: '-',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'TINYBLOB',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'BLOB',
              default: null,
              group: false,
              children: []
            },
            {
              name: '-',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'ENUM',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'SET',
              default: null,
              group: false,
              children: []
            }
          ]
        },
        {
          name: 'Spatial',
          default: null,
          group: true,
          children: [
            {
              name: 'GEOMETRY',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'POINT',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'LINESTRING',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'POLYGON',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'MULTIPOINT',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'MULTILINESTRING',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'MULTIPOLYGON',
              default: null,
              group: false,
              children: []
            },
            {
              name: 'GEOMETRYCOLLECTION',
              default: null,
              group: false,
              children: []
            }
          ]
        },
        {
          name: 'JSON',
          default: null,
          group: true,
          children: [
            {
              name: 'JSON',
              default: null,
              group: false,
              children: []
            }
          ]
        }
      ]
    }),

    /**
     * Retrieve databases and tables (if db provided)
     */
    created () {
      let $vm = this

      // Assign conn
      $vm.conn = $vm.$parent.connection.ref

      // Db set in connection
      if ($vm.$route.query.db !== null && $vm.$route.query.db !== '') {
        $vm.use($vm.$route.query.db)
      }

      // Retrieve databases
      $vm.conn.query('SHOW DATABASES')
      .then(res => {
        $vm.databases = res.map(row => row.Database)
      })
      .catch(err => {
        $vm.$swal('Error', 'Error retrieving databases: ' + err.message, 'error')
      })

      // Retrieve tables, if database selected
      if ($vm.conn.config.database !== null) {
        $vm.getTables()
      }
    },

    methods: {
      /**
       * Change database
       *
       * @param {string} db
       */
      use (db) {
        let $vm = this

        $vm.conn.changeUser({database: db}, () => {
          $vm.$parent.db = db
          $vm.getTables()

          // Clear cache current table
          $vm.table = null
          $vm.$parent.table = null
          $vm.columns = {}
          $vm.primary_key = null
          $vm.foreign_keys = {}
          // Clear all db cache
          $vm.tables_primary_keys = {}
          $vm.tables_foreign_keys = {}
          $vm.tables_columns = {}

          $vm.foreignKeys(db)
          .then(res => {
            for (let fk of res) {
              let { 0: fromTable, 1: fromCol } = fk.foreign_key.split('.')
              let { 0: toTable, 1: toCol } = fk.references.split('.')

              // Create empty array if doesn't exists
              if (!$vm.tables_foreign_keys.hasOwnProperty(fromTable)) {
                $vm.tables_foreign_keys[fromTable] = {}
              }

              $vm.tables_foreign_keys[fromTable][fromCol] = {
                table: toTable,
                col: toCol
              }
            }
          })
          .catch(err => $vm.$swal('Error', 'Error retrieving foreign keys: ' + err.message, 'error'))

          $vm.$router.replace('/connected')
        })
      },

      /**
       * Show table structure
       *
       * @param {string} table
       * @param {bool} redirect
       */
      structure (table, redirect = true) {
        let $vm = this

        $vm.table = table
        $vm.$parent.table = table
        $vm.columns = []

        $vm.foreign_keys = $vm.tables_foreign_keys[table] || {}

        // Retrieve from history
        if ($vm.tables_columns.hasOwnProperty(table)) {
          $vm.columns = $vm.tables_columns[table]
          $vm.primary_key = $vm.tables_primary_keys[table]
          if (redirect) {
            $vm.$router.push('/connected')
          }
          return
        }

        // Otherwise, retrieve structure
        $vm.conn.query('DESCRIBE ' + table)
        .then(res => {
          // Parse fields
          $vm.columns = res.map(f => {
            let fUnsigned = false
            let fLen = null

            let fData = f.Type.split(' ') // Ex: int(11) unsigned
            let fType = fData[0]

            // Check if unsigned
            if (fData.length > 1) {
              fUnsigned = true
            }

            // Retrieve length
            if (fType.indexOf('(') > -1) {
              let fLenMatch = fType.match(/\(([^)]+)\)/g)
              if (fLenMatch !== null) {
                fLen = fLenMatch[0].replace('(', '').replace(')', '')
                fType = fType.replace('(' + fLen + ')', '')
              }
            }

            // Check if primary key (used in Content)
            if (f.Key === 'PRI') {
              $vm.primary_key = f.Field
              $vm.tables_primary_keys[table] = f.Field
            }

            return {
              name: f.Field,
              type: fType.toUpperCase(),
              len: fLen,
              unsigned: fUnsigned,
              null: f.Null === 'YES',
              key: f.Key,
              default: f.Default,
              extra: f.Extra
            }
          })

          // Cache fields
          $vm.tables_columns[table] = $vm.columns

          if (redirect) {
            $vm.$router.push('/connected')
          }
        })
        .catch(err => $vm.$swal('Error', 'Error retrieving table info: ' + err.message, 'error'))
      },

      /**
       * Retrieve database tables
       */
      getTables () {
        let $vm = this

        $vm.conn.query('SHOW TABLES')
        .then(res => {
          $vm.tables = res.map(row => row['Tables_in_' + $vm.$parent.db])
        })
        .catch(err => {
          $vm.$swal('Error', 'Error retrieving tables: ' + err.message, 'error')
        })
      },

      /**
       * Return filtered tables
       *
       * @param {Array} tables
       * @return {Array}
       */
      searchTable (tables) {
        return tables.filter(t => t.toLowerCase().indexOf(this.filterTable.toLowerCase()) > -1)
      },

      /**
       * Retrieve all foreign keys
       *
       * @param {string} db
       *
       * @return {Promise}
       */
      foreignKeys (db) {
        return this.conn.query(`select
            concat(table_name, '.', column_name) as 'foreign_key',  
            concat(referenced_table_name, '.', referenced_column_name) as 'references'
        from
            information_schema.key_column_usage
        where
          table_schema = '${db}'
        and
            referenced_table_name is not null`)
      },

      /**
       * Handle context menu open
       *
       * @param {object} conn
       */
      onCtxOpen (conn) {
        this.ctxData = conn
      }
    }
  }
</script>

<style lang="scss">
main {
  overflow: auto;
  padding-right: 0.75rem !important;

  .main-table {
    height: 60%;

    &.full {
      height: calc(100% - 100px);
    }
  }

  .secondary-table {
    height: 40%;
  }

  .scrollable {
    overflow: auto;
  }

  .scrollable-x {
    overflow-x: auto;
  }

  table.table {
    margin-bottom: 0;

    th {
      white-space: nowrap;
      border-bottom: 1px solid #d6d6d6 !important;
      position: relative;

      &:not(:first-child)::before {
        position: absolute;
        width: 1px;
        height: 60%;
        top: 20%;
        left: 0;
        background: #e5e5e5;
        content: '';
      }
    }

    tr.is-active {
      background: $primary !important;
      * {
        color: #fff !important;
      }

      option,
      optgroup {
        color: #363636 !important;
      }

      .select::after {
        border-color: #fff !important;
      }
    }
  }

  table th,
  table td {
    vertical-align: middle !important;
    border: none !important;

    &.has-input {
      padding-top: 0 !important;
      padding-bottom: 0 !important;

      input {
        padding-top: 0.25em;
        padding-bottom: 0.25em;
      }
    }

    input[type="checkbox"] {
      cursor: pointer;
    }

    input[type="text"] {
      display: table;
      text-overflow: ellipsis;

      &.smaller {
        width: 60px;
      }
    }

    .select {
      &:hover::after {
        border-color: $primary;
      }

      select {
        background: transparent;
        border: none;
        outline: none;
        border-radius: 0;

        &:focus {
          border: none;
          box-shadow: none;
        }
      }
    }
  }

  .table.is-narrow td {
    height: 30px;
    position: relative;

    input,
    .select,
    .select select {
      max-width: 100%;
      height: 100%;
    }

    &.has-icon {
      input,
      .select {
        max-width: calc(100% - 40px);
        display: inline-block;
      }

      i.fa {
        font-size: 14px;
        opacity: .5;
        position: absolute;
        cursor: pointer;
        top: 6px;
        right: 0;
      }
    }
  }
}

aside.menu ul.menu-list li a {
  img {
    position: relative;
    top: 2px;
    margin-right: 5px;
  }

  &:hover {
    color: #fff;
  }
}
</style>
