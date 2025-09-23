### Spcp.js
```js
const path = require("node:path");
const fs = require("node:fs/promises");
const process = require("process");

const pluginName = "Spcp";

const DEBUG = process.env.SPCP_DEBUG === "true";

function debugLog(...args) {
  if (DEBUG) {
    console.log("[Spcp Debug]:", ...args);
  }
}

function camelCase(str) {
  function capitalizeFirstLetter(string) {
    return string.charAt(0).toUpperCase() + string.slice(1);
  }
  const tokens = str.split(/[^a-zA-Z0-9]|((?<=\D)\B(?=\d))|((?<=\d)\B(?=[a-zA-Z]))/g).filter(Boolean);
  return tokens[0] + tokens.slice(1).map(capitalizeFirstLetter).join("");
}

function sortObjectKeysAlphabeticallyRecursive(obj) {
  if (typeof obj !== "object" || obj === null || Array.isArray(obj)) {
    return obj;
  }
  const keys = Object.keys(obj);
  keys.sort((a, b) => {
    return a.toLowerCase().localeCompare(b.toLowerCase());
  });
  const sortedObj = {};
  for (const key of keys) {
    const value = obj[key];
    if (typeof value === "object" && value !== null && !Array.isArray(value)) {
      sortedObj[key] = sortObjectKeysAlphabeticallyRecursive(value);
    } else {
      sortedObj[key] = value;
    }
  }
  return sortedObj;
}

class Spcp {
  constructor() {
    this.template = path.resolve(process.cwd(), "templates", "pages.ts.template");
    this.output = path.resolve(process.cwd(), "constants", "pages.ts");
    this.appDir = path.resolve(process.cwd(), "app");
    this.cache = null;
    this.timer = null;
    this.runing = false;
    this.isClosing = false;
    this.watcher = null;
    this.prettierConfig = null;
    this.split = this.split.bind(this);
    this.getPages = this.getPages.bind(this);
    this.emitFile = this.emitFile.bind(this);
    this.build = this.build.bind(this);
    this.setupWatcher = this.setupWatcher.bind(this);
    this.setupSignalHandlers = this.setupSignalHandlers.bind(this);
    this.apply = this.apply.bind(this);
  }

  split(page) {
    let result = [];
    for (const segment of page.split("/")) {
      if (segment.startsWith("(") || segment.startsWith("[") || segment === "app" || segment === "page.tsx") {
        continue;
      }
      result.push(segment);
    }
    return result;
  }

  async getPages() {
    try {
      debugLog("开始获取页面文件");
      const { glob } = require("glob");
      const pages = await glob(["app/**/page.tsx"], {
        ignore: ["node_modules/**", "**/\\[...rest\\]/**", "**/_*/**"],
        posix: true,
        cwd: process.cwd(), // 确保在正确的工作目录中搜索
      });
      debugLog(`找到 ${pages.length} 个页面文件`);
      return pages.sort();
    } catch (error) {
      debugLog("获取页面文件错误:", error);
      return [];
    }
  }

  toTree(pages) {
    return pages.reduce(
      (acc, page) => {
        const segments = this.split(page);
        let current = acc;
        for (const segment of segments) {
          if (!current[segment]) {
            current[camelCase(segment)] = {
              asPath: `${current.asPath ?? ""}/${segment}`,
            };
          }
          current = current[segment];
        }
        return acc;
      },
      { root: { asPath: "/" } },
    );
  }

  async emitFile(tree) {
    const template = await fs.readFile(this.template, "utf-8");
    await fs
      .access(path.dirname(this.output), fs.constants.F_OK)
      .catch(() => fs.mkdir(path.dirname(this.output), { recursive: true }))
      .then(async () => {
        let data = template.replace("$PAGES$", JSON.stringify(sortObjectKeysAlphabeticallyRecursive(tree), void 0, 2));
        try {
          const prettier = require("prettier");
          if (!this.prettierConfig) {
            this.prettierConfig = await prettier.resolveConfig();
          }
          data = await prettier.format(data, { ...this.prettierConfig, parser: "typescript" });
        } catch (_) {
          debugLog("Cannot find module 'prettier'");
        }
        return await fs.writeFile(this.output, data);
      })
      .catch(debugLog);
  }

  async build(_, callback) {
    const pages = await this.getPages();
    if (this.cache !== JSON.stringify(pages)) {
      this.emitFile(this.toTree(pages));
      this.cache = JSON.stringify(pages);
    } else {
      debugLog("use cache");
    }
    callback?.();
  }

  setupWatcher() {
    const chokidar = require("chokidar");
    debugLog("初始化监听器");

    this.watcher = chokidar.watch(this.appDir, {
      ignoreInitial: false,
      atomic: true,
      followSymlinks: true,
      ignored: /node_modules/,
    });
    const debouncedBuild = async (filePath) => {
      if (!filePath.endsWith("page.tsx")) return;
      // 防抖处理
      if (this.timer) clearTimeout(this.timer);
      this.timer = setTimeout(async () => {
        try {
          await this.build();
          debugLog("build successed");
        } catch (error) {
          debugLog("构建错误", error);
        }
      }, 300);
    };
    this.watcher
      .on("add", debouncedBuild)
      .on("unlink", debouncedBuild)
      .on("error", (error) => {
        debugLog("监听器错误", error);
      });
    this.setupSignalHandlers();
  }

  setupSignalHandlers() {
    try {
      const closeWatcher = async (signal) => {
        if (this.isClosing) {
          debugLog("正在退出...");
          return;
        }

        this.isClosing = true;
        debugLog(`接收到 ${signal} 信号，正在关闭监视器...`);

        try {
          if (this.timer) {
            clearTimeout(this.timer);
            this.timer = null;
          }

          if (this.watcher) {
            await this.watcher.close();
            debugLog("监视器已关闭");
          }

          process.exit(0);
        } catch (error) {
          debugLog("关闭监视器时出错:", error);
          process.exit(1);
        }
      };

      process.on("SIGINT", () => closeWatcher("SIGINT"));
      process.on("SIGTERM", () => closeWatcher("SIGTERM"));
      process.on("SIGHUP", () => closeWatcher("SIGHUP"));
      process.on("exit", (code) => closeWatcher(`exit ${code}`));
    } catch (error) {
      debugLog("设置信号处理程序时出错:", error);
    }
  }

  apply(compiler) {
    if (compiler.options.name !== "client" || this.isRuning) {
      return;
    }
    this.isRuning = true;

    if (process.env.NODE_ENV === "production") {
      compiler.hooks.environment.tap(pluginName, this.build);
    }
    if (process.env.NODE_ENV === "development") {
      this.setupWatcher();
    }
  }
}
module.exports = Spcp;
if (require.main === module) {
  const spcp = new Spcp();
  if (process.env.NODE_ENV === "development") {
    spcp.setupWatcher();
  } else {
    spcp.build();
  }
}
```

### template
```ts
import { locales } from "~i18n/config";

const PageRoute = $PAGES$ as const;
type PageRouteType = typeof PageRoute;
type WithFunction<T> = {
  [K in keyof T]: T[K] extends object
    ? WithFunction<T[K]> & { toString: () => string; valueOf: () => string; is: (path: string) => boolean }
    : T[K];
};
function patched(obj: PageRouteType): WithFunction<PageRouteType> {
  const s = [obj];
  while (s.length) {
    const o: any = s.pop(),
      keys = Object.keys(o);
    Object.freeze(o);
    for (const key of keys) {
      let v;
      if (key !== "asPath" && typeof (v = o[key])?.asPath === "string") {
        v.valueOf = function () {
          return this.asPath;
        };
        v.toString = function () {
          return this.asPath;
        };
        v.is = function (path: string) {
          return (
            this.asPath === path ||
            locales.some((locale) => {
              return path === `/${locale}${this.asPath === "/" ? "" : this.asPath}`;
            })
          );
        };
        s.push(v);
      }
    }
  }
  return obj as WithFunction<PageRouteType>;
}
const Pages = patched(PageRoute);
export default Pages;
```
