<template>
    <PanelItem :index="index" :field="field">
        <template #value>
            <div class="overflow-hidden">
                <iframe :id="iframeId" sandbox="allow-scripts allow-same-origin" importance="low" width="100%" height="800"></iframe>
            </div>
        </template>
    </PanelItem>
</template>

<script>
export default {
    props: ['index', 'resource', 'resourceName', 'resourceId', 'field'],

    beforeCreate() {
        this.iframeId = `${this.field.uniqueKey}-iframe`;
    },

    mounted() {
        let iframe = document.getElementById(this.iframeId);
        this.setIframeContent(iframe, this.field.html || 'HTML is not set for preview, use <code>Unlayer::html()</code>');
    },

    methods: {
        /**
         * @param {HTMLIFrameElement} iframe
         * @param {string} htmlContent
         */
        setIframeContent: function (iframe, htmlContent) {
            iframe.srcdoc = '<div>' + htmlContent + '</div>';
        },
    },
}
</script>
