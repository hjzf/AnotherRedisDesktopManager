<template>
<div class="connection-menu-title">
  <div class="connection-opt-icons">
    <!-- right menu operate icons -->
    <i :title="$t('message.redis_status')"
      class="connection-right-icon fa fa-home"
      :style="{ color: client ? '#7cad7c' : ''}"
      @click.stop.prevent="openStatus">
    </i>
    <i :title="$t('message.redis_console')"
      class="connection-right-icon fa fa-terminal font-weight-bold"
      @click.stop.prevent="openCli">
    </i>
    <i :title="$t('message.refresh_connection')"
      class='connection-right-icon el-icon-refresh font-weight-bold'
      @click.stop.prevent="refreshConnection">
    </i>

    <!-- more operate menu -->
    <el-dropdown
      class='connection-menu-more'
      placement='bottom-start'
      :show-timeout=100
      :hide-timeout=300>
      <i class="connection-right-icon el-icon-menu" @click.stop></i>
      <el-dropdown-menu class='connection-menu-more-ul' slot="dropdown">


        <el-dropdown-item @click.native='closeConnection'>
          <span><i class='more-operate-ico fa fa-power-off'></i>&nbsp;{{ $t('message.close_connection') }}</span>
        </el-dropdown-item>
        <el-dropdown-item @click.native='showEditConnection'>
          <span><i class='more-operate-ico el-icon-edit-outline'></i>&nbsp;{{ $t('message.edit_connection') }}</span>
        </el-dropdown-item>
        <el-dropdown-item @click.native='deleteConnection'>
          <span><i class='more-operate-ico el-icon-delete'></i>&nbsp;{{ $t('message.del_connection') }}</span>
        </el-dropdown-item>
        <el-dropdown-item @click.native='duplicateConnection'>
          <span><i class='more-operate-ico fa fa-clone'></i>&nbsp;{{ $t('message.duplicate_connection') }}</span>
        </el-dropdown-item>

        <!-- menu color picker -->
        <el-tooltip placement="right" effect="light">
          <el-color-picker
            slot='content'
            v-model="menuColor"
            @change='changeColor'
            :predefine="['#f56c6c', '#F5C800', '#409EFF', '#85ce61', '#c6e2ff']">
          </el-color-picker>

          <el-dropdown-item divided>
            <span><i class='more-operate-ico fa fa-bookmark-o'></i>&nbsp;{{ $t('message.mark_color') }}</span>
          </el-dropdown-item>
        </el-tooltip>

        <el-dropdown-item @click.native='memoryAnalisys'>
          <span><i class='more-operate-ico fa fa-table'></i>&nbsp;{{ $t('message.memory_analysis') }}</span>
        </el-dropdown-item>
        <el-dropdown-item @click.native='slowLog'>
          <span><i class='more-operate-ico fa fa-hourglass-start'></i>&nbsp;{{ $t('message.slow_log') }}</span>
        </el-dropdown-item>
        <el-dropdown-item @click.native='importKeys' divided>
          <span><i class='more-operate-ico el-icon-download'></i>&nbsp;{{ $t('message.import') }} Key</span>
        </el-dropdown-item>
        <el-dropdown-item @click.native='execFileCMDS'>
          <span><i class='more-operate-ico fa fa-file-code-o'></i>&nbsp;{{ $t('message.import') }} CMD</span>
        </el-dropdown-item>
        <el-dropdown-item @click.native='flushDB'>
          <span><i class='more-operate-ico fa fa-exclamation-triangle'></i>&nbsp;{{ $t('message.flushdb') }}</span>
        </el-dropdown-item>

      </el-dropdown-menu>
    </el-dropdown>
  </div>
  <div :title="connectionTitle()" class="connection-name">{{config.connectionName}}
    <!-- <i v-if="client" style="position: absolute; left: 2px; bottom: 5px; width: 8px; height: 8px; border-radius: 4px; background-color: green;"></i> -->
  </div>

  <!-- edit connection dialog -->
  <NewConnectionDialog
    editMode='true'
    :config='config'
    @editConnectionFinished='editConnectionFinished'
    ref='editConnectionDialog'>
  </NewConnectionDialog>
</div>
</template>

<script type="text/javascript">
import storage from '@/storage.js';
import { remote } from 'electron';
import NewConnectionDialog from '@/components/NewConnectionDialog';
import splitargs from '@qii404/redis-splitargs';

export default {
  data() {
    return {
      menuColor: '#409EFF',
    };
  },
  props: ['config', 'client'],
  components: { NewConnectionDialog },
  created() {
    this.$bus.$on('duplicateConnection', (newConfig) => {
      // not self
      if (this.config.name !== newConfig.name) {
        return;
      }

      this.showEditConnection();
    });
  },
  methods: {
    connectionTitle() {
      const { config } = this;
      const sep = '-----------';
      const lines = [
        config.connectionName,
        sep,
        `${this.$t('message.host')}: ${config.host}`,
        `${this.$t('message.port')}: ${config.port}`,
      ];

      config.username && lines.push(`${this.$t('message.username')}: ${config.username}`);
      config.separator && lines.push(`${this.$t('message.separator')}: "${config.separator}"`);

      if (config.connectionReadOnly) {
        lines.push(`${sep}\nREADONLY`);
      }
      if (config.sshOptions) {
        lines.push(`${sep}\nSSH:`);
        lines.push(`  ${this.$t('message.host')}: ${config.sshOptions.host}`);
        lines.push(`  ${this.$t('message.port')}: ${config.sshOptions.port}`);
        lines.push(`  ${this.$t('message.username')}: ${config.sshOptions.username}`);
      }
      if (config.cluster) {
        lines.push(`${sep}\nCLUSTER`);
      }
      if (config.sentinelOptions) {
        lines.push(`${sep}\nSENTINEL:`);
        lines.push(`  ${this.$t('message.master_group_name')}: ${config.sentinelOptions.masterName}`);
      }

      return lines.join('\n');
    },
    refreshConnection() {
      this.$emit('refreshConnection');
    },
    showEditConnection() {
      // connection is cloesd, do not display confirm
      if (!this.client) {
        return this.$refs.editConnectionDialog.show();
      }

      this.$confirm(
        this.$t('message.close_to_edit_connection'),
        { type: 'warning' },
      ).then(() => {
        this.$bus.$emit('closeConnection', this.config.connectionName);
        this.$refs.editConnectionDialog.show();
      }).catch(() => {});
    },
    closeConnection() {
      this.$confirm(
        this.$t('message.close_to_connection'),
        { type: 'warning' },
      ).then(() => {
        this.$bus.$emit('closeConnection', this.config.connectionName);
      }).catch(() => {});
    },
    editConnectionFinished(newConfig) {
      this.$bus.$emit('refreshConnections');
    },
    duplicateConnection() {
      // empty key\order , just as a new connection
      const newConfig = {
        ...this.config,
        key: undefined,
        order: undefined,
        connectionName: undefined,
      };

      storage.addConnection(newConfig);

      this.$bus.$emit('refreshConnections');
      // 100ms after connection list is ready
      setTimeout(() => {
        this.$bus.$emit('duplicateConnection', newConfig);
      }, 100);
    },
    deleteConnection() {
      this.$confirm(
        this.$t('message.confirm_to_delete_connection'),
        { type: 'warning' },
      ).then(() => {
        storage.deleteConnection(this.config);
        this.$bus.$emit('refreshConnections');

        this.$message.success({
          message: this.$t('message.delete_success'),
          duration: 1000,
        });
      }).catch(() => {});
    },
    openStatus() {
      if (!this.client) {
        // open Connections.vue menu
        this.$parent.$parent.$parent.$refs.connectionMenu.open(this.config.connectionName);
        // open connection
        this.$parent.$parent.$parent.openConnection();
      } else {
        this.$bus.$emit('openStatus', this.client, this.config.connectionName);
      }
    },
    openCli() {
      // open cli before connection opened
      if (!this.client) {
        // open Connections.vue menu
        this.$parent.$parent.$parent.$refs.connectionMenu.open(this.config.connectionName);
        // open connection
        this.$parent.$parent.$parent.openConnection(() => {
          this.$bus.$emit('openCli', this.client, this.config.connectionName);
        });
      } else {
        this.$bus.$emit('openCli', this.client, this.config.connectionName);
      }
    },
    memoryAnalisys() {
      if (!this.client) {
        return;
      }

      this.$bus.$emit('memoryAnalysis', this.client, this.config.connectionName);
    },
    slowLog() {
      if (!this.client) {
        return;
      }

      this.$bus.$emit('slowLog', this.client, this.config.connectionName);
    },
    importKeys() {
      remote.dialog.showOpenDialog(remote.getCurrentWindow(), {
        properties: ['openFile'],
      }).then((reply) => {
        if (reply.canceled) {
          return;
        }

        const succ = [];
        const fail = [];
        let count = 0;

        const rl = require('readline').createInterface({
          input: require('fs').createReadStream(reply.filePaths[0]),
        });

        rl.on('line', (line) => {
          let [key, content, ttl] = line.split(',');

          if (!key || !content) {
            return;
          }

          count++;

          // show notify in first time
          if (count === 1) {
            this.$notify.success({
              message: this.$createElement('p', { ref: 'importKeysNotify' }, ''),
              duration: 0,
            });
          }

          key = Buffer.from(key, 'hex');
          content = Buffer.from(content, 'hex');
          ttl = ttl > 0 ? ttl : 0;

          // fix #1213, REPLACE can be used in Redis>=3.0
          this.client.callBuffer('RESTORE', key, ttl, content, 'REPLACE').then((reply) => {
            // reply == 'OK'
            succ.push(key);
          }).catch((e) => {
            fail.push(key);
          }).finally(() => {
            this.$set(this.$refs.importKeysNotify,
              'innerHTML',
              `Succ: ${succ.length}, Fail: ${fail.length}`);
          });
        });

        rl.on('close', () => {
          if (count === 0) {
            return this.$message.error('File parse failed.');
          }

          (count > 10000) && this.$message.success({
            message: this.$t('message.import_success'),
            duration: 800,
          });

          // refresh keu list
          this.$bus.$emit('refreshKeyList', this.client);
        });
      });
    },
    execFileCMDS() {
      remote.dialog.showOpenDialog(remote.getCurrentWindow(), {
        properties: ['openFile'],
      }).then((reply) => {
        if (reply.canceled) {
          return;
        }

        const succ = [];
        const fail = [];
        let count = 0;

        const rl = require('readline').createInterface({
          input: require('fs').createReadStream(reply.filePaths[0]),
        });

        rl.on('line', (line) => {
          const paramsArr = splitargs(line, true);

          if (!paramsArr || !paramsArr.length) {
            return;
          }

          count++;

          // show notify in first time
          if (count === 1) {
            this.$notify.success({
              message: this.$createElement('p', { ref: 'importCMDNotify' }, ''),
              duration: 0,
            });
          }

          this.client.callBuffer(...paramsArr).then((reply) => {
            succ.push(line);
          }).catch((e) => {
            fail.push(line);
          }).finally(() => {
            this.$set(this.$refs.importCMDNotify,
              'innerHTML',
              `Succ: ${succ.length}, Fail: ${fail.length}`
            );
          });
        });

        rl.on('close', () => {
          if (count === 0) {
            return this.$message.error('File parse failed.');
          }

          (count > 10000) && this.$message.success({
            message: this.$t('message.import_success'),
            duration: 800,
          });

          // refresh key list
          this.$bus.$emit('refreshKeyList', this.client);
        });
      });
    },
    flushDB() {
      if (!this.client) {
        return;
      }

      const preDB = this.client.condition ? this.client.condition.select : 0;
      const inputTxt = 'y';
      const placeholder = this.$t('message.flushdb_prompt', { txt: inputTxt });

      this.$prompt(this.$t('message.confirm_flush_db', { db: preDB }), {
        inputValidator: value => ((value == inputTxt) ? true : placeholder),
        inputPlaceholder: placeholder,
      })
        .then((value) => {
          this.client.flushdb().then((reply) => {
            if (reply == 'OK') {
              this.$message.success({
                message: this.$t('message.delete_success'),
                duration: 1000,
              });

              this.refreshConnection();
            }
          }).catch((e) => { this.$message.error(e.message); });
        })
        .catch((e) => {});
    },
    changeColor(color) {
      this.$emit('changeColor', color);
    },
  },
};
</script>

<style type="text/css">
  .connection-menu-title {
    margin-left: -20px;
  }

  .connection-menu .connection-name {
    margin-right: 115px;
    padding-left: 6px;
    padding-right: 6px;
    word-break: keep-all;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    font-weight: bold;
    font-size: 1.04em;
  }
  .connection-menu .connection-opt-icons {
    /*width: 30px;*/
    /*float: right;
    margin-right: 28px;*/
    position: absolute;
    right: 25px;
    top: -2px;
  }
  .connection-menu .connection-right-icon {
    display: inline-block;
    font-size: 1.16em;
    /*font-weight: bold;*/
    padding: 3px;
    margin-right: -4px;
    transition: background 0.2s;
  }
  .connection-menu .connection-right-icon:hover {
    /*color: #85878a;*/
    background: #dcdee0;
    border-radius: 3px;
  }
  .dark-mode .connection-menu .connection-right-icon:hover {
    background: #58707b;
  }

  /*fix more operation btn icon vertical-center*/
  .connection-menu-more {
    vertical-align: baseline;
  }
  /*more operation ul>ico*/
  .connection-menu-more-ul .more-operate-ico {
    width: 13px;
    text-align: center;
  }

  .font-weight-bold {
    font-weight: bold;
  }
</style>
