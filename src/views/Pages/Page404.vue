<template>
    <div class="app flex-row align-items-center">
        <div class="container">
            <b-row class="justify-content-center">
                <b-col md="6">
                    <div class="clearfix">
                        <h1 class="float-left display-3 mr-4">404</h1>
                        <h4 class="pt-3">Oops! You're lost.</h4>
                        <p class="text-muted">The page you are looking for was not found.<br/><router-link :to="{ name: 'Home' }">Go to homepage.</router-link></p>
                    </div>
                </b-col>
            </b-row>
        </div>
    </div>
</template>

<script>
    export default {
        name: 'Page404'
    }
</script>
