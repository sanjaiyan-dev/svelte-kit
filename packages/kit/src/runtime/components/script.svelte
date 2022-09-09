<script context="module">
	const ScriptCache = new Map()
const LoadCache = new Set()

const ignoreProps = [
    'onLoad',
    'onReady',
    'dangerouslySetInnerHTML',
    'children',
    'onError',
    'strategy',
]

const loadScript = (props) => {
    const {
        src,
        id,
        onLoad = () => { },
        onReady = null,
        dangerouslySetInnerHTML,
        children = '',
        strategy = 'afterInteractive',
        onError,
    } = props

    const cacheKey = id || src

    // Script has already loaded
    if (cacheKey && LoadCache.has(cacheKey)) {
        return
    }

    // Contents of this script are already loading/loaded
    if (ScriptCache.has(src)) {
        LoadCache.add(cacheKey)
        // Execute onLoad since the script loading has begun
        ScriptCache.get(src).then(onLoad, onError)
        return
    }

    /** Execute after the script first loaded */
    const afterLoad = () => {
        // Run onReady for the first time after load event
        if (onReady) {
            onReady()
        }
        // add cacheKey to LoadCache when load successfully
        LoadCache.add(cacheKey)
    }

    const el = document.createElement('script')

    const loadPromise = new Promise((resolve, reject) => {
        el.addEventListener('load', function (e) {
            resolve()
            if (onLoad) {
                onLoad.call(this, e)
            }
            afterLoad()
        })
        el.addEventListener('error', function (e) {
            reject(e)
        })
    }).catch(function (e) {
        if (onError) {
            onError(e)
        }
    })

    if (dangerouslySetInnerHTML) {
        el.innerHTML = dangerouslySetInnerHTML.__html || ''

        afterLoad()
    } else if (children) {
        el.textContent =
            typeof children === 'string'
                ? children
                : Array.isArray(children)
                    ? children.join('')
                    : ''

        afterLoad()
    } else if (src) {
        el.src = src
        // do not add cacheKey into LoadCache for remote script here
        // cacheKey will be added to LoadCache when it is actually loaded (see loadPromise above)

        ScriptCache.set(src, loadPromise)
    }

    for (const [k, value] of Object.entries(props)) {
        if (value === undefined || ignoreProps.includes(k)) {
            continue
        }

        const attr = DOMAttributeNames[k] || k.toLowerCase()
        el.setAttribute(attr, value)
    }

    el.setAttribute('data-svelte-script', strategy)

    document.body.appendChild(el)
}

function handleClientScriptLoad(props) {
    const { strategy = 'afterInteractive' } = props
    if (strategy === 'lazyOnload') {
        window.addEventListener('load', () => {
            requestIdleCallback(() => loadScript(props))
        })
    } else {
        loadScript(props)
    }
}

function loadLazyScript(props) {
    if (document.readyState === 'complete') {
        requestIdleCallback(() => loadScript(props))
    } else {
        window.addEventListener('load', () => {
            requestIdleCallback(() => loadScript(props))
        })
    }
}

function addBeforeInteractiveToCache() {
    const scripts = [
        ...document.querySelectorAll('[data-svelte-script="beforeInteractive"]'),
        ...document.querySelectorAll('[data-svelte-script="beforePageRender"]'),
    ]
    scripts.forEach((script) => {
        const cacheKey = script.id || script.getAttribute('src')
        LoadCache.add(cacheKey)
    })
}

function initScriptLoader(scriptLoaderItems) {
    scriptLoaderItems.forEach(handleClientScriptLoad)
    addBeforeInteractiveToCache()
}

</script>

<script>
	let name = 'world';
</script>

<h1>Hello {name}!</h1>