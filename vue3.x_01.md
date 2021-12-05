# vue 3.0

在学习源码的过程中，有很多复杂的逻辑，当时可能理解了，但是随着时间的流逝，经常会很多的重要的知识点会被遗忘，主要是因为源码的思想一方面很复杂，有很多的算法思想在里面，理解起来会有困难，另一方面是因为，我们阅读的源码在平时开发的过程中使用不到。不用，所以经常会忘记。所以我想写一些读源码的过程中、记录一些笔记，可以方便日后在复习这些内容的时候。做些提示

## Vue 入口

Vue3 的项目下划分了很多的模块。入口是在vue目录下

```javascript
// packages/vue/src/index.ts
const compileCache: Record<string, RenderFunction> = Object.create(null)
function compileToFunction(
  template: string | HTMLElement,
  options?: CompilerOptions
): RenderFunction {
    
  // 省略。。。
    
  const key = template
  const cached = compileCache[key] // 这里先试图从缓存中取
  if (cached) {
    return cached
  }

	// 跳过非关键代码。。。
  const { code } = compile(
    template,
    extend(
      {
        hoistStatic: true,
        onError(err: CompilerError) {
          if (__DEV__) {
            const message = `Template compilation error: ${err.message}`
            const codeFrame =
              err.loc &&
              generateCodeFrame(
                template as string,
                err.loc.start.offset,
                err.loc.end.offset
              )
            warn(codeFrame ? `${message}\n${codeFrame}` : message)
          } else {
            /* istanbul ignore next */
            throw err
          }
        }
      },
      options
    )
  )

  // The wildcard import results in a huge object with every export
  // with keys that cannot be mangled, and can be quite heavy size-wise.
  // In the global build we know `Vue` is available globally so we can avoid
  // the wildcard object.
  const render = (__GLOBAL__
    ? new Function(code)()
    : new Function('Vue', code)(runtimeDom)) as RenderFunction

  // mark the function as runtime compiled
  ;(render as InternalRenderFunction)._rc = true
	// 设置缓存
  return (compileCache[key] = render)
}

registerRuntimeCompiler(compileToFunction)

export { compileToFunction as compile }
export * from '@vue/runtime-dom'
```

可以看到在这个文件里主要是定义了一个***compileToFunction***的函数，然后调用***registerRuntimeCompiler***把***compileToFunction***进行了注册，然后将***compileToFunction*** 导出为**compile**,最后导出**@vue/runtime-dom**中所有的模块（在后面的内容中再分析）

```javascript
// packages/runtime-core/src/component.ts
export function registerRuntimeCompiler(_compile: any) {
  // 就是将compileToFunction 设置为components的compile。
  //components 这个文件在后面分析
  compile = _compile
}
```

***compileToFunction***主要逻辑是调用**@vue/compiler-dom** 里的comile ，将组件的templete 编译后生成一个***code***字符串，然后利用这个字符串 **new Function ** (即**render**)，挂载compileCache上做一个缓存，并返回这个render。即***compileToFunction*** 接受template 并返回一个render,(即组件的render函数，这里主要是把组件模板编译后生成对应的***vnode***）

我们再来看看compile里做了什么

```javascript
// packages/compiler-dom/src/index.ts
export function compile(
  template: string,
  options: CompilerOptions = {}
): CodegenResult {
  return baseCompile(
    template,
    extend({}, parserOptions, options, {
      nodeTransforms: [
        // ignore <script> and <tag>
        // this is not put inside DOMNodeTransforms because that list is used
        // by compiler-ssr to generate vnode fallback branches
        ignoreSideEffectTags,
        ...DOMNodeTransforms,
        ...(options.nodeTransforms || [])
      ],
      directiveTransforms: extend(
        {},
        DOMDirectiveTransforms,
        options.directiveTransforms || {}
      ),
      transformHoist: __BROWSER__ ? null : stringifyStatic
    })
  )
}
```

```javascript
// packages/compiler-core/src/compile.ts
export function baseCompile(
  template: string | RootNode,
  options: CompilerOptions = {}
): CodegenResult {
	// 跳过。。。
  
  const ast = isString(template) ? baseParse(template, options) : template
  const [nodeTransforms, directiveTransforms] = getBaseTransformPreset(
    prefixIdentifiers
  )
  transform(
    ast,
    extend({}, options, {
      prefixIdentifiers,
      nodeTransforms: [
        ...nodeTransforms,
        ...(options.nodeTransforms || []) // user transforms
      ],
      directiveTransforms: extend(
        {},
        directiveTransforms,
        options.directiveTransforms || {} // user transforms
      )
    })
  )

  return generate(
    ast,
    extend({}, options, {
      prefixIdentifiers
    })
  )
}
```

可以看到，这里主要逻辑是把template 经过 **baseParse** -> **ast** -> **transform **- > **generate** -> **code字符串**

### 1. parse

把传进来的template 解析成ast抽象语法树

```javascript
// packages/compiler-core/src/parse.ts
export function baseParse(
  content: string,
  options: ParserOptions = {}
): RootNode {
  const context = createParserContext(content, options) // 创建解析的上下文对象,javascript对象记录一些options信息,template 设置为context里的source,originalSource
  const start = getCursor(context) // 生成记录解析过程的游标信息  column, line, offset 
  return createRoot(// 生成并返回 root 根节点
    parseChildren(context, TextModes.DATA, []),// 解析子节点，作为 root 根节点的 children 属性
    getSelection(context, start)
  )
}

// packages/compiler-core/src/ast.ts
export function createRoot( // 返回一个javascript对象
  children: TemplateChildNode[],
  loc = locStub
): RootNode {
  return {
    type: NodeTypes.ROOT,
    children,
    helpers: [],
    components: [],
    directives: [],
    hoists: [],
    imports: [],
    cached: 0,
    temps: 0,
    codegenNode: undefined,
    loc
  }
}
// packages/compiler-core/src/parse.ts
function parseChildren(
  context: ParserContext,
  mode: TextModes,
  ancestors: ElementNode[]
): TemplateChildNode[] {
  const parent = last(ancestors)
  const ns = parent ? parent.ns : Namespaces.HTML
  const nodes: TemplateChildNode[] = [] // 用来装解析后的node
	// 这里是解析tempalte的逻辑，从头开始解析，通过判断调用 parseComment 、parseBogusComment、parseText等方法生成对应的node,然后调用pushNode 放进 nodes数组
  while (!isEnd(context, mode, ancestors)) {
    __TEST__ && assert(context.source.length > 0)
    const s = context.source // 即前面的template
    let node: TemplateChildNode | TemplateChildNode[] | undefined = undefined

    if (mode === TextModes.DATA || mode === TextModes.RCDATA) {
      if (!context.inVPre && startsWith(s, context.options.delimiters[0])) {
        // '{{'
        node = parseInterpolation(context, mode)
      } else if (mode === TextModes.DATA && s[0] === '<') {
        // https://html.spec.whatwg.org/multipage/parsing.html#tag-open-state
        if (s.length === 1) {
          emitError(context, ErrorCodes.EOF_BEFORE_TAG_NAME, 1)
        } else if (s[1] === '!') {
          // https://html.spec.whatwg.org/multipage/parsing.html#markup-declaration-open-state
          if (startsWith(s, '<!--')) {
            node = parseComment(context)
          } else if (startsWith(s, '<!DOCTYPE')) {
            // Ignore DOCTYPE by a limitation.
            node = parseBogusComment(context)
          } else if (startsWith(s, '<![CDATA[')) {
            if (ns !== Namespaces.HTML) {
              node = parseCDATA(context, ancestors)
            } else {
              emitError(context, ErrorCodes.CDATA_IN_HTML_CONTENT)
              node = parseBogusComment(context)
            }
          } else {
            emitError(context, ErrorCodes.INCORRECTLY_OPENED_COMMENT)
            node = parseBogusComment(context)
          }
        } else if (s[1] === '/') {
          // https://html.spec.whatwg.org/multipage/parsing.html#end-tag-open-state
          if (s.length === 2) {
            emitError(context, ErrorCodes.EOF_BEFORE_TAG_NAME, 2)
          } else if (s[2] === '>') {
            emitError(context, ErrorCodes.MISSING_END_TAG_NAME, 2)
            advanceBy(context, 3)
            continue
          } else if (/[a-z]/i.test(s[2])) {
            emitError(context, ErrorCodes.X_INVALID_END_TAG)
            parseTag(context, TagType.End, parent)
            continue
          } else {
            emitError(
              context,
              ErrorCodes.INVALID_FIRST_CHARACTER_OF_TAG_NAME,
              2
            )
            node = parseBogusComment(context)
          }
        } else if (/[a-z]/i.test(s[1])) {
          node = parseElement(context, ancestors)
        } else if (s[1] === '?') {
          emitError(
            context,
            ErrorCodes.UNEXPECTED_QUESTION_MARK_INSTEAD_OF_TAG_NAME,
            1
          )
          node = parseBogusComment(context)
        } else {
          emitError(context, ErrorCodes.INVALID_FIRST_CHARACTER_OF_TAG_NAME, 1)
        }
      }
    }
    // 如果上述的情况解析完毕后，没有创建对应的节点，则当做文本来解析
    if (!node) {
      node = parseText(context, mode)
    }

    if (isArray(node)) {
      for (let i = 0; i < node.length; i++) {
        pushNode(nodes, node[i])
      }
    } else {
      pushNode(nodes, node)
    }
  }
	//下面是对空格的一些处理
  // Whitespace management for more efficient output
  // (same as v2 whitespace: 'condense')
  let removedWhitespace = false
  if (mode !== TextModes.RAWTEXT && mode !== TextModes.RCDATA) {
    for (let i = 0; i < nodes.length; i++) {
      const node = nodes[i]
      if (!context.inPre && node.type === NodeTypes.TEXT) {
        if (!/[^\t\r\n\f ]/.test(node.content)) {
          const prev = nodes[i - 1]
          const next = nodes[i + 1]
          // If:
          // - the whitespace is the first or last node, or:
          // - the whitespace is adjacent to a comment, or:
          // - the whitespace is between two elements AND contains newline
          // Then the whitespace is ignored.
          if (
            !prev ||
            !next ||
            prev.type === NodeTypes.COMMENT ||
            next.type === NodeTypes.COMMENT ||
            (prev.type === NodeTypes.ELEMENT &&
              next.type === NodeTypes.ELEMENT &&
              /[\r\n]/.test(node.content))
          ) {
            removedWhitespace = true
            nodes[i] = null as any
          } else {
            // Otherwise, condensed consecutive whitespace inside the text
            // down to a single space
            node.content = ' '
          }
        } else {
          node.content = node.content.replace(/[\t\r\n\f ]+/g, ' ')
        }
      }
      // also remove comment nodes in prod by default
      if (
        !__DEV__ &&
        node.type === NodeTypes.COMMENT &&
        !context.options.comments
      ) {
        removedWhitespace = true
        nodes[i] = null as any
      }
    }
    if (context.inPre && parent && context.options.isPreTag(parent.tag)) {
      // remove leading newline per html spec
      // https://html.spec.whatwg.org/multipage/grouping-content.html#the-pre-element
      const first = nodes[0]
      if (first && first.type === NodeTypes.TEXT) {
        first.content = first.content.replace(/^\r?\n/, '')
      }
    }
  }
	// 最后返回解析后的nodes数组（ast）
  return removedWhitespace ? nodes.filter(Boolean) : nodes
}
```

关于parse可以阅读这篇[文章](https://juejin.cn/post/6959421748416774180),以更好的理解

### 2. transform

transform阶段主要是做静态提升 hoistStatic

#### 静态提升

当 Vue 的编译器在编译过程中，发现了一些不会变的节点或者属性，就会给这些节点打上标记。然后编译器在生成代码字符串的过程中，会发现这些静态的节点，并提升它们，将他们序列化成字符串，以此减少编译及渲染成本。有时可以跳过一整棵树

比如

```javascript
<div>
  <div>
    <span class="foo"></span>
    <span class="foo"></span>
    <span class="foo"></span>
    <span class="foo"></span>
    <span class="foo"></span>
  </div>
</div>

如果没有静态提升:
const { createVNode: _createVNode, openBlock: _openBlock, createBlock: _createBlock } = Vue

return function render(_ctx, _cache) {
  return (_openBlock(), _createBlock("div", null, [
    _createVNode("div", null, [
      _createVNode("span", { class: "foo" }),
      _createVNode("span", { class: "foo" }),
      _createVNode("span", { class: "foo" }),
      _createVNode("span", { class: "foo" }),
      _createVNode("span", { class: "foo" })
    ])
  ]))
}

静态提升后：
const { createVNode: _createVNode, createStaticVNode: _createStaticVNode, openBlock: _openBlock, createBlock: _createBlock } = Vue

const _hoisted_1 = /*#__PURE__*/_createStaticVNode("<div><span class=\"foo\"></span><span class=\"foo\"></span><span class=\"foo\"></span><span class=\"foo\"></span><span class=\"foo\"></span></div>", 1)

return function render(_ctx, _cache) {
  return (_openBlock(), _createBlock("div", null, [
    _hoisted_1
  ]))
}

```

[这里有一个vue编译的在线地址](https://link.segmentfault.com/?url=https%3A%2F%2Freurl.cc%2FXWDGvR)

```javascript
export function transform(root: RootNode, options: TransformOptions) {
  const context = createTransformContext(root, options)// 创建context javascript对象 内部保存一些属性和一些方法
  traverseNode(root, context) // 从根开始遍历，进行转换
  if (options.hoistStatic) {// 如果编译选项中打开了 hoistStatic 开关，则进行静态提升
    hoistStatic(root, context)
  }
  if (!options.ssr) {
    createRootCodegen(root, context)
  }
  // finalize meta information
  root.helpers = [...context.helpers.keys()]
  root.components = [...context.components]
  root.directives = [...context.directives]
  root.imports = context.imports
  root.hoists = context.hoists
  root.temps = context.temps
  root.cached = context.cached
}


export function traverseNode(
  node: RootNode | TemplateChildNode,
  context: TransformContext
) {
  context.currentNode = node
  // apply transform plugins
  const { nodeTransforms } = context
  const exitFns = []
  for (let i = 0; i < nodeTransforms.length; i++) {
		// 跳过。。。
  }

  switch (node.type) {
    case NodeTypes.COMMENT:
      if (!context.ssr) {
        // inject import for the Comment symbol, which is needed for creating
        // comment nodes with `createVNode`
        context.helper(CREATE_COMMENT)
      }
      break
    case NodeTypes.INTERPOLATION:
      // no need to traverse, but we need to inject toString helper
      if (!context.ssr) {
        context.helper(TO_DISPLAY_STRING)
      }
      break

    // for container types, further traverse downwards
    case NodeTypes.IF:
      for (let i = 0; i < node.branches.length; i++) {
        traverseNode(node.branches[i], context)
      }
      break
    case NodeTypes.IF_BRANCH:
    case NodeTypes.FOR:
    case NodeTypes.ELEMENT:
    case NodeTypes.ROOT:
      traverseChildren(node, context) // node即是createRoot创建的javascript对象
      break
  }

  // exit transforms
  context.currentNode = node
  let i = exitFns.length
  while (i--) {
    exitFns[i]()
  }
}


export function traverseChildren(
  parent: ParentNode,// parent 初始应该是root节点
  context: TransformContext
) {
  let i = 0
  const nodeRemoved = () => {
    i--
  }
  for (; i < parent.children.length; i++) { // parent.children 是 parse阶段，返回的nodes[] 数组
    const child = parent.children[i]
    if (isString(child)) continue
    context.parent = parent
    context.childIndex = i
    context.onNodeRemoved = nodeRemoved
    traverseNode(child, context) // 又执行traverseNode
  }
}

// 静态提升 packages/compiler-core/src/transforms/hoistStatic.ts
export function hoistStatic(root: RootNode, context: TransformContext) {
  walk(
    root,
    context,
    // Root node is unfortunately non-hoistable due to potential parent
    // fallthrough attributes.
    isSingleElementRoot(root, root.children[0])
  )
}


export const enum ConstantTypes {
  NOT_CONSTANT = 0,
  CAN_SKIP_PATCH,
  CAN_HOIST,
  CAN_STRINGIFY
}


function walk(
  node: ParentNode,
  context: TransformContext,
  doNotHoistNode: boolean = false
) {
  let hasHoistedNode = false
  // Some transforms, e.g. transformAssetUrls from @vue/compiler-sfc, replaces
  // static bindings with expressions. These expressions are guaranteed to be
  // constant so they are still eligible for hoisting, but they are only
  // available at runtime and therefore cannot be evaluated ahead of time.
  // This is only a concern for pre-stringification (via transformHoist by
  // @vue/compiler-dom), but doing it here allows us to perform only one full
  // walk of the AST and allow `stringifyStatic` to stop walking as soon as its
  // stringficiation threshold is met.
  let canStringify = true

  const { children } = node // children 即parse阶段返回的node数组
  for (let i = 0; i < children.length; i++) { 
    const child = children[i]
    // only plain elements & text calls are eligible for hoisting.
    if (
      child.type === NodeTypes.ELEMENT &&
      child.tagType === ElementTypes.ELEMENT
    ) {
       // 如果不允许被提升，则赋值 constantType NOT_CONSTANT 不可被提升的标记
    	// 否则调用 getConstantType 获取子节点的静态类型
      const constantType = doNotHoistNode
        ? ConstantTypes.NOT_CONSTANT
        : getConstantType(child, context)
      if (constantType > ConstantTypes.NOT_CONSTANT) {
        if (constantType < ConstantTypes.CAN_STRINGIFY) {
          canStringify = false
        }
        if (constantType >= ConstantTypes.CAN_HOIST) { // 可以提升
           // 则将子节点的 codegenNode 属性的 patchFlag 标记为 HOISTED 可提升
          ;(child.codegenNode as VNodeCall).patchFlag =
            PatchFlags.HOISTED + (__DEV__ ? ` /* HOISTED */` : ``)
          child.codegenNode = context.hoist(child.codegenNode!)
          hasHoistedNode = true
          continue
        }
      } else {
        // node may contain dynamic children, but its props may be eligible for
        // hoisting.
        // 节点可能包含动态的子节点，但是它的 props 属性也可能能被合法提升
        const codegenNode = child.codegenNode!
        if (codegenNode.type === NodeTypes.VNODE_CALL) {
          const flag = getPatchFlag(codegenNode)
          if ( // 如果不存在 flag，或者 flag 是文本类型
        			// 并且该节点 props 的 constantType 值判断出可以被提升
            (!flag ||
              flag === PatchFlags.NEED_PATCH ||
              flag === PatchFlags.TEXT) &&
            getGeneratedPropsConstantType(child, context) >=
              ConstantTypes.CAN_HOIST
          ) {
            const props = getNodeProps(child)
            if (props) {
              codegenNode.props = context.hoist(props)
            }
          }
        }
      }
    } else if (child.type === NodeTypes.TEXT_CALL) {// 如果节点类型为 TEXT_CALL，则同样进行检查，逻辑与前面一致
      const contentType = getConstantType(child.content, context)
      if (contentType > 0) {
        if (contentType < ConstantTypes.CAN_STRINGIFY) {
          canStringify = false
        }
        if (contentType >= ConstantTypes.CAN_HOIST) {
          child.codegenNode = context.hoist(child.codegenNode)
          hasHoistedNode = true
        }
      }
    }

    // walk further
    if (child.type === NodeTypes.ELEMENT) {
      // 如果子节点的 tagType 是组件，则继续遍历子节点
      const isComponent = child.tagType === ElementTypes.COMPONENT
      if (isComponent) {
        context.scopes.vSlot++
      }
      walk(child, context)
      if (isComponent) {
        context.scopes.vSlot--
      }
    } else if (child.type === NodeTypes.FOR) {
      // Do not hoist v-for single child because it has to be a block
       // 但是如果 v-for 的节点中是只有一个子节点，则不能被提升
      walk(child, context, child.children.length === 1)
    } else if (child.type === NodeTypes.IF) {
      for (let i = 0; i < child.branches.length; i++) {
        // Do not hoist v-if single child because it has to be a block
        // 如果子节点是 v-if 类型，判断它所有的分支情况
        walk(
          child.branches[i],
          context,
          child.branches[i].children.length === 1
        )
      }
    }
  }

  if (canStringify && hasHoistedNode && context.transformHoist) {
    context.transformHoist(children, context, node);// 如果可以进行静态提升则调用context中的transformHoist进行转换
  }
}
```



关于transform 可以阅读[这篇文章](https://juejin.cn/post/6961170732743131173)

### 3. generate

经过compile后组件的template会编译成code 字符串，然后通过Function 的构造函数生成render函数

```javascript
export interface CodegenResult {
  code: string
  preamble: string
  ast: RootNode
  map?: RawSourceMap
}

export function generate(
  ast: RootNode,
  options: CodegenOptions & {
    onContextCreated?: (context: CodegenContext) => void
  } = {}
): CodegenResult {
  const context = createCodegenContext(ast, options)
  if (options.onContextCreated) options.onContextCreated(context)
  const {
    mode,
    push,
    prefixIdentifiers,
    indent,
    deindent,
    newline,
    scopeId,
    ssr
  } = context

  const hasHelpers = ast.helpers.length > 0
  const useWithBlock = !prefixIdentifiers && mode !== 'module'
  const genScopeId = !__BROWSER__ && scopeId != null && mode === 'module'
  const isSetupInlined = !__BROWSER__ && !!options.inline

  // preambles
  // in setup() inline mode, the preamble is generated in a sub context
  // and returned separately.
  const preambleContext = isSetupInlined
    ? createCodegenContext(ast, options)
    : context
  if (!__BROWSER__ && mode === 'module') {
    genModulePreamble(ast, preambleContext, genScopeId, isSetupInlined)
  } else {
    genFunctionPreamble(ast, preambleContext)
  }

  // enter render function
  const functionName = ssr ? `ssrRender` : `render`
  const args = ssr ? ['_ctx', '_push', '_parent', '_attrs'] : ['_ctx', '_cache']
  if (!__BROWSER__ && options.bindingMetadata && !options.inline) {
    // binding optimization args
    args.push('$props', '$setup', '$data', '$options')
  }
  const signature =
    !__BROWSER__ && options.isTS
      ? args.map(arg => `${arg}: any`).join(',')
      : args.join(', ')

  if (genScopeId && !isSetupInlined) {
    // root-level _withId wrapping is no longer necessary after 3.0.8 and is
    // a noop, it's only kept so that code compiled with 3.0.8+ can run with
    // runtime < 3.0.8.
    // TODO: consider removing in 3.1
    push(`const ${functionName} = ${PURE_ANNOTATION}${WITH_ID}(`)
  }
  if (isSetupInlined || genScopeId) {
    push(`(${signature}) => {`)
  } else {
    push(`function ${functionName}(${signature}) {`)
  }
  indent()

  if (useWithBlock) {
    push(`with (_ctx) {`)
    indent()
    // function mode const declarations should be inside with block
    // also they should be renamed to avoid collision with user properties
    if (hasHelpers) {
      push(
        `const { ${ast.helpers
          .map(s => `${helperNameMap[s]}: _${helperNameMap[s]}`)
          .join(', ')} } = _Vue`
      )
      push(`\n`)
      newline()
    }
  }

  // generate asset resolution statements
  if (ast.components.length) {
    genAssets(ast.components, 'component', context)
    if (ast.directives.length || ast.temps > 0) {
      newline()
    }
  }
  if (ast.directives.length) {
    genAssets(ast.directives, 'directive', context)
    if (ast.temps > 0) {
      newline()
    }
  }
  if (ast.temps > 0) {
    push(`let `)
    for (let i = 0; i < ast.temps; i++) {
      push(`${i > 0 ? `, ` : ``}_temp${i}`)
    }
  }
  if (ast.components.length || ast.directives.length || ast.temps) {
    push(`\n`)
    newline()
  }

  // generate the VNode tree expression
  if (!ssr) {
    push(`return `)
  }
  if (ast.codegenNode) {
    genNode(ast.codegenNode, context)
  } else {
    push(`null`)
  }

  if (useWithBlock) {
    deindent()
    push(`}`)
  }

  deindent()
  push(`}`)

  if (genScopeId && !isSetupInlined) {
    push(`)`)
  }

  return {
    ast,
    code: context.code,
    preamble: isSetupInlined ? preambleContext.code : ``,
    // SourceMapGenerator does have toJSON() method but it's not in the types
    map: context.map ? (context.map as any).toJSON() : undefined
  }
}
```

通过generate 生成的code用于后面的生成render函数

可以阅读这篇[文章](https://juejin.cn/post/6964566021449449508)

