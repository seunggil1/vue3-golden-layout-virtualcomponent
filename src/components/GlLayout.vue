<template>
    <div style="position: relative">
        <div ref="GLRoot" style="position: absolute; width: 100%; height: 100%">
            <!-- Root dom for Golden-Layout manager -->
        </div>
        <div style="position: absolute; width: 100%; height: 100%">
            <gl-component
                v-for="pair in AllComponents"
                :key="pair[0]"
                :ref="GlcKeyPrefix + pair[0]"
            >
                <component :is="pair[1]"></component>
            </gl-component>
        </div>
    </div>
</template>

<script>
import {
    onMounted,
    ref,
    markRaw,
    readonly,
    defineExpose,
    defineAsyncComponent,
    defineProps,
    nextTick,
    getCurrentInstance,
} from "vue";
import {
    ComponentContainer,
    Json,
    LayoutConfig,
    RowOrColumnItemConfig,
    StackItemConfig,
    ComponentItemConfig,
    ResolvedComponentItemConfig,
    LogicalZIndex,
    VirtualLayout,
    ResolvedLayoutConfig,
} from "golden-layout";
import GlComponent from "@/components/GlComponent.vue";

export default {
    name : "GlLayout",
    props : {
        glcPath: String,
    },

    components: {
        GlComponent,
    },

    setup(props) {
        const GLRoot = ref(undefined);
        let GLayout;
        const GlcKeyPrefix = readonly(ref("glc_"));

        const MapComponents = new Map();
        const AllComponents = ref(new Map());
        const UnusedIndexes = [];
        let CurIndex = 0;
        let GlBoundingClientRect;

        const instance = getCurrentInstance();

        /*******************
         * Method
         *******************/
        /** @internal */
        const addComponent = (componentType, title) => {
            const glc = markRaw(
                defineAsyncComponent(
                    () => import(props.glcPath + componentType + ".vue")
                )
            );

            let index = CurIndex;
            if (UnusedIndexes.length > 0) 
                index = UnusedIndexes.pop();
            else 
                CurIndex++;

            AllComponents.value.set(index, glc);

            return index;
        };

        const addGLComponent = async (componentType, title) => {
            if (componentType.length == 0)
                throw new Error("addGLComponent: Component's type is empty");

            const index = addComponent(componentType, title);

            await nextTick(); // wait 1 tick for vue to add the dom

            GLayout.addComponent(componentType, { refId: index }, title);
        };

        const loadGLLayout = async ( layoutConfig ) => {
            GLayout.clear();
            AllComponents.value.clear();

            const config = (layoutConfig.resolved ? LayoutConfig.fromResolved(layoutConfig) : layoutConfig);

            let contents = [config.root.content];

            let index = 0;
            while (contents.length > 0) {
                const content = contents.shift();
                for (let itemConfig of content) {
                    if (itemConfig.type == "component") {
                        index = addComponent(
                            itemConfig.componentType,
                            itemConfig.title
                        );
                        console.log(typeof itemConfig.componentState);
                        if (typeof itemConfig.componentState == "object")
                            itemConfig.componentState["refId"] = index;
                        else itemConfig.componentState = { refId: index };
                    } else if (itemConfig.content.length > 0) {
                        contents.push(itemConfig.content);
                    }
                }
            }

            await nextTick(); // wait 1 tick for vue to add the dom

            GLayout.loadLayout(config);
        };

        const getLayoutConfig = () => {
            return GLayout.saveLayout();
        };

        /*******************
         * Mount
         *******************/
        onMounted(() => {
            if (GLRoot.value == undefined)
                throw new Error("Golden Layout can't find the root DOM!");

            const onResize = () => {
                const dom = GLRoot.value;
                let width = dom ? dom.offsetWidth : 0;
                let height = dom ? dom.offsetHeight : 0;
                GLayout.setSize(width, height);
            };

            window.addEventListener("resize", onResize, { passive: true });

            const handleBeforeVirtualRectingEvent = (count) => {
                GlBoundingClientRect = ( GLRoot.value ).getBoundingClientRect();
            };

            const handleContainerVirtualRectingRequiredEvent = (container, width, height) => {
                const component = MapComponents.get(container);
                if (!component || !component?.glc) {
                    throw new Error( "handleContainerVirtualRectingRequiredEvent: Component not found" );
                }

                const containerBoundingClientRect = container.element.getBoundingClientRect();
                const left = containerBoundingClientRect.left - GlBoundingClientRect.left;
                const top = containerBoundingClientRect.top - GlBoundingClientRect.top;
                component.glc.setPosAndSize(left, top, width, height);
            };

            const handleContainerVirtualVisibilityChangeRequiredEvent = ( container, visible ) => {
                const component = MapComponents.get(container);
                if (!component || !component?.glc) {
                    throw new Error("handleContainerVirtualVisibilityChangeRequiredEvent: Component not found");
                }

                component.glc.setVisibility(visible);
            };

            const handleContainerVirtualZIndexChangeRequiredEvent = (container,logicalZIndex,defaultZIndex) => {
                const component = MapComponents.get(container);
                if (!component || !component?.glc) {
                    throw new Error("handleContainerVirtualZIndexChangeRequiredEvent: Component not found");
                }

                component.glc.setZIndex(defaultZIndex);
            };

            const bindComponentEventListener = (container,itemConfig) => {
                let refId = -1;
                if (itemConfig && itemConfig.componentState) {
                    refId = (itemConfig.componentState).refId;
                } else {
                    throw new Error("bindComponentEventListener: component's ref id is required");
                }

                const ref = GlcKeyPrefix.value + refId;
                const component = instance?.refs[ref];

                MapComponents.set(container, { refId: refId, glc: component });

                container.virtualRectingRequiredEvent = (container, width, height) =>
                    handleContainerVirtualRectingRequiredEvent(container, width, height);

                container.virtualVisibilityChangeRequiredEvent = (container, visible) =>
                    handleContainerVirtualVisibilityChangeRequiredEvent(container,visible);

                container.virtualZIndexChangeRequiredEvent = (
                    container,
                    logicalZIndex,
                    defaultZIndex
                ) =>
                    handleContainerVirtualZIndexChangeRequiredEvent(
                        container,
                        logicalZIndex,
                        defaultZIndex
                    );

                return {
                    component,
                    virtual: true,
                };
            };

            const unbindComponentEventListener = ( container ) => {
                const component = MapComponents.get(container);
                if (!component || !component?.glc) {
                    throw new Error("handleUnbindComponentEvent: Component not found");
                }

                MapComponents.delete(container);
                AllComponents.value.delete(component.refId);
                UnusedIndexes.push(component.refId);
            };

            GLayout = new VirtualLayout(
                GLRoot.value,
                bindComponentEventListener,
                unbindComponentEventListener
            );

            GLayout.beforeVirtualRectingEvent = handleBeforeVirtualRectingEvent;
        });

        return {
            GLRoot,
            GLayout,
            GlcKeyPrefix,
            MapComponents,
            AllComponents,
            UnusedIndexes,
            CurIndex,
            GlBoundingClientRect,
            instance,

            addGLComponent,
            loadGLLayout,
            getLayoutConfig
        }
    }
}
</script>