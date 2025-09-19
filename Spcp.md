```js
const path = require("node:path");
const fs = require("node:fs/promises");
const { glob } = require("glob");
const prettier = require("prettier");

const pluginName = "Spcp";

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
    this.innerId = Math.random().toString(36).slice(2);
    this.template = path.resolve(__dirname, "..", "templates", "pages.ts.template");
    this.output = path.resolve(__dirname, "..", "constants", "pages.ts");
    this.timer = null;
    this.runing = false;
    this.getPages = this.getPages.bind(this);
    this.split = this.split.bind(this);
    this.toTree = this.toTree.bind(this);
    this.apply = this.apply.bind(this);
    this.emitFile = this.emitFile.bind(this);
    this.build = this.build.bind(this);
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
    return await glob(["app/**/page.tsx"], {
      ignore: ["node_modules/**", "**/\\[...rest\\]/**", "**/_*/**"],
    });
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
        const options = await prettier.resolveConfig();
        data = await prettier.format(data, { ...options, parser: "typescript" });
        return await fs.writeFile(this.output, data);
      })
      .catch((e) => {
        console.error("[Spcp emitFile error]: ", e);
      });
  }

  async build() {
    const pages = await this.getPages();
    this.emitFile(this.toTree(pages));
  }
  apply(compiler) {
    if (compiler.options.name !== "client") {
      return;
    }
    console.log("[Spcp apply]");

    compiler.hooks.beforeRun.tapPromise(pluginName, this.build);
    if (process.env.NODE_ENV === "development") {
      compiler.hooks.watchRun.tapPromise(pluginName, this.build);
    }
  }
}
module.exports = Spcp;
if (require.main === module) {
  const spcp = new Spcp();
  spcp.getPages().then((pages) => {
    // console.log(spcp.toTree(pages));
    spcp.emitFile(spcp.toTree(pages));
  });
}

```
