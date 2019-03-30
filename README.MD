const bindViewToData = (el, data) => {

    /*create VM tree Node from real dom
     * index is only id of every node
     */
    let index = 0
    const createNode = (el, nodes = []) => {
        let childs = el.children
        childs = Array.from(childs)
        if (!childs.length) {
            return
        } else {
            childs.forEach((child, idx) => {
                index++
                nodes.push({
                    index,
                    name: child.tagName,
                    varText: child.innerText,
                    text: child.innerText,
                    children: []
                })
                createNode(child, nodes[idx].children)
            })
        }
        return nodes
    }

    /*compile variable into VM node*/
    const compileHTML = (nodes, data) => {
        nodes.forEach(node => {
            node = doCompile(node, data)
            if (node.children.length) {
                compileHTML(node.children, data)
            }
        })
        return nodes
    }
    const doCompile = (node, data) => {
        let inner = node.varText
        let REGC = /{{([^{^}]+)}}/g
        node.text = inner.replace(REGC, (val, match, index) => {
            //remove space add quote
            match = match.replace(/[\s\+\'\"]+/g, ' ')
            // case {{ aa + bb }} , key means aa and bb, i means 0 and 3
            return match.replace(/[a-zA-Z_]+/g, (key, i) => {
                //link the dataKey to node index,when dataKey change 
                // when dataKey change , only change the indexed node
                varTable = createVarTable(node.index, key)
                return data[key]
            })
        })
        return node
    }
    const compileOneNode = (node, data) => {
        return doCompile(node, data)
    }


    // create varTable to VM by nodes idx
    let varTable = {}
    const createVarTable = (idx, value) => {
        if (!varTable[value]) {
            varTable[value] = []
        }
        varTable[value].push(idx)
        return varTable
    }

    // update dom's data after compile
    const createDom = (el, vnodes) => {
        for (let i = 0; i < el.length; i++) {
            el[i].innerText = vnodes[i].text
            if (vnodes[i].children.length) {
                createDom(el[i].children, vnodes[i].children)
            } else {
                return
            }
        }

    }

    //listen input button ,when it change , change the data
    const beginListenInput = () => {
        let enumInput = ["firstName", "lastName", 'age', 'name']
        for (let inputStr of enumInput) {
            let dom = document.getElementById(inputStr)
            dom.addEventListener('input', (val) => {
                data[inputStr] = val.target.value
            })
        }
    }
    //defineProperty for all [key,value] in data
    const setDataModel = (data, nodes) => {
        let dataKVs = Object.entries(data)
        dataKVs.forEach((dataKV, sidx) => {
            let dataKey = dataKV[0],
                dataVal = dataKV[1]
            defineSet(dataKey, dataVal, nodes)
        })
    }
    const defineSet = (dataKey, dataVal, nodes) => {
        Object.defineProperty(data, dataKey, {
            get: function() {
                return this['_' + dataKey] || dataVal;
            },
            /* 1. because in set , we use value to set data[dataKey],will use set method
             * will call set method  for ever,so we use this['_' + dataKey] to set value
             * 2. to  improve efficiency we use setTimeout to update dom
             * the macro tasklist will done in the last
             */
            set: function(value) {
                let targetVmIndex = []
                let vNodeIndexArr = varTable[dataKey]
                vNodeIndexArr = [...new Set(vNodeIndexArr)]
                this['_' + dataKey] = value
                vNodeIndexArr.forEach(targetIdx => {
                    setTimeout(updateDom(el, nodes, targetIdx), 0)
                })
            }
        });
    }
    /* 
     * update dom layer by layer , becuase the node's index is unique 
     * so if find in this layer ,we will finish find ,do not need to the next layer
     * */
    const updateDom = (el, nodes, targetIdx) => {
        let findDomThisLayer = false
        nodes.forEach((node, idx) => {
            if (node.index == targetIdx) {
                let vnode = compileOneNode(node, data)
                el.children[idx].innerText = vnode.text
                findDomThisLayer = true
            }
        })
        if (!findDomThisLayer) {
            nodes.forEach((node, idx) => {
                updateDom(el.children[idx], nodes[idx].children, targetIdx)
            })
        }
    }
    let nodes = createNode(el)
    let vnodes = compileHTML(nodes, data)
    createDom(el.children, vnodes)
    beginListenInput()
    setDataModel(data, nodes)
}
