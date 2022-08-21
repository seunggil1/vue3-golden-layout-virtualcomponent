<template>
    <div>
        <h2>{{ code }}</h2>
        <h2>{{ a }}</h2>
        <button class="favorite styled"
                type="button"
                onclick="click()"
                >
            Add to favorites
        </button>
    </div>
</template>

<script>

export default {
    props : {
        code : String
    },
    setup () {
        const a = ref(0);

        const click = () => {
            a.value++;
        }
        return {
            a,
            click
        }
    }
}
</script>

<style lang="scss" scoped>

</style>