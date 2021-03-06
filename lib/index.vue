<template>
  <script :id="id" type="text/plain"></script>
</template>

<script>
// 一个简单的事件订阅发布的实现,取代原始Event对象,提升IE下的兼容性
class LoadEvent {
  constructor() {
    this.listeners = {};
  }
  on(eventName, callback) {
    this.listeners[eventName] === undefined ? (this.listeners[eventName] = []) : '';
    this.listeners[eventName].push(callback);
  }
  emit(eventName) {
    this.listeners[eventName] && this.listeners[eventName].forEach((callback) => callback());
  }
}
export default {
  name: 'VueUeditorWrap',
  data() {
    return {
      isLoading:false,
      id:
        'editor' +
        Math.random()
          .toString()
          .slice(-10),
      editor: null,
      isReady: false, // 实例是否已经ready
      readyValue: '', // ready之后给编辑器设置的值
      defaultConfig: {
        UMEDITOR_HOME_URL: '/g/vendors/js/umeditor/',
        enableAutoSave: false
      }
    };
  },
  props: {
    value: {
      type: String,
      default: ''
    },
    control: {
      type: Object,
      default: function() {
        return {};
      }
    },
    config: {
      type: Object,
      default: function() {
        return {};
      }
    },
    init: {
      type: Function,
      default: function() {
        return () => {};
      }
    },
    destroy: Boolean,
    uploadType: {
      type: String,
      require: true
    }
  },
  computed: {
    mixedConfig() {
      return Object.assign({}, this.defaultConfig, this.config);
    }
  },
  methods: {
    // 添加自定义按钮
    registerButton: ({ name, icon, tip, handler, UE = window.UM }) => {
      UE.registerUI(name, (editor, name) => {
        editor.registerCommand(name, {
          execCommand: () => {
            handler(editor, name);
          }
        });
        const btn = new UE.ui.Button({
          name,
          title: tip,
          cssRules: `background-image: url(${icon}) !important;background-size: cover;`,
          onclick() {
            editor.execCommand(name);
          }
        });
        editor.addListener('selectionchange', () => {
          const state = editor.queryCommandState(name);
          if (state === -1) {
            btn.setDisabled(true);
            btn.setChecked(false);
          } else {
            btn.setDisabled(false);
            btn.setChecked(state);
          }
        });
        return btn;
      });
    },
    // 实例化编辑器之前-JS依赖检测
    _beforeInitEditor(value) {
      this.readyValue = value;
      const load = () => {
        if(!this.isLoading){
          this.load();
        }
      }
      // 准确判断ueditor.config.js和ueditor.all.js是否均已加载 仅加载完ueditor.config.js时UE对象和UEDITOR_CONFIG对象也存在,仅加载完ueditor.all.js时UEDITOR_CONFIG对象也存在,但为空对象
      !!window.UM && !!window.UM.getEditor ? this._initEditor(value) : load();
    },
    load(){
      this.isLoading = true;
      this._loadScripts().then(() => {
        this._initEditor(this.readyValue)
      });
    },
    // 实例化编辑器
    _initEditor(value) {
      this.$nextTick(() => {
        console.log('next=>' + value);
        this.init();
        this.editor = UM.getEditor(this.id, this.mixedConfig);
        this.editor.ready(() => {
          this.editor.execCommand('serverparam', 'type', this.uploadType);
          this.isReady = true;
          this.$emit('ready', this.editor);
           console.log('readyValue=>' + this.readyValue);
          this.editor.setContent(this.readyValue);
          this.editor.addListener('contentChange', () => {
            this.$emit('input', this.editor.getContent());
          });
        });
      });
    },
    // 动态添加JS依赖
    _loadScripts() {
      window.UMEDITOR_HOME_URL = this.mixedConfig.UMEDITOR_HOME_URL;
      // 确保多个实例同时渲染时不会重复创建SCRIPT标签
      if (window.loadEnv) {
        return new Promise((resolve, reject) => {
          window.loadEnv.on('scriptsLoaded', function() {
            resolve();
          });
        });
      } else {
        window.loadEnv = new LoadEvent();
        return new Promise((resolve, reject) => {
          // 如果在其他地方只引用ueditor.all.min.js，在加载ueditor.config.js之后仍需要重新加载ueditor.all.min.js，所以必须确保ueditor.config.js已加载
          this._loadJquery()
            .then(() => this._loadConfig())
            .then(() => this._loadCore())
            .then(() => this._loadLang())
            .then(() => {
              window.loadEnv.emit('scriptsLoaded');
              resolve();
            });
        });
      }
    },
    _loadLang() {
      return new Promise((resolve, reject) => {
        let configScript = document.createElement('script');
        configScript.type = 'text/javascript';
        configScript.src = this.mixedConfig.UMEDITOR_HOME_URL + 'lang/zh-cn/zh-cn.js';
        document.getElementsByTagName('head')[0].appendChild(configScript);
        configScript.onload = function() {
          resolve();
        };
      });
    },
    _loadJquery() {
      return new Promise((resolve, reject) => {
        if (!!window.$) {
          resolve();
          return;
        }
        let configScript = document.createElement('script');
        configScript.type = 'text/javascript';
        configScript.src = '/g/vendors/js/jquery-1.11.2.js';
        document.getElementsByTagName('head')[0].appendChild(configScript);
        configScript.onload = function() {
          if (!!window.$) {
            resolve();
          } else {
            console && console.error('加载jquery失败,请检查您的配置地址UMEDITOR_HOME_URL填写是否正确!');
          }
        };
      });
    },
    _loadConfig() {
      return new Promise((resolve, reject) => {
        if (!!window.UM) {
          resolve();
          return;
        }
        let configScript = document.createElement('script');
        configScript.type = 'text/javascript';
        configScript.src = this.mixedConfig.UMEDITOR_HOME_URL + 'umeditor.config.js';
        document.getElementsByTagName('head')[0].appendChild(configScript);
        configScript.onload = function() {
          resolve();
        };
      });
    },
    _loadCore() {
      return new Promise((resolve, reject) => {
        if (!!window.UM) {
          resolve();
          return;
        }
        let coreScript = document.createElement('script');
        coreScript.type = 'text/javascript';
        coreScript.src = this.mixedConfig.UMEDITOR_HOME_URL + 'umeditor.js';
        document.getElementsByTagName('head')[0].appendChild(coreScript);
        coreScript.onload = function() {
          if (!!window.UM) {
            resolve();
          } else {
            console && console.error('umeditor.js失败,请检查您的配置地址UMEDITOR_HOME_URL填写是否正确!');
          }
        };
      });
    },
    // 设置内容
    _setContent(value) {
      console.log('setcontent=>' + value)
      if (this.isReady) {
        value === this.editor.getContent() || this.editor.setContent(value);
      } else {
        this.readyValue = value;
      }
    }
  },
  beforeDestroy() {
    if (this.destroy && this.editor && this.editor.destroy) this.editor.destroy();
  },
  // v-model语法糖实现
  watch: {
    value: {
      handler(value) {
        this.editor ? this._setContent(value) : this._beforeInitEditor(value);
        this.$emit('change', value, this.control);
      },
      immediate: true
    }
  }
};
</script>
