<template>
    <div class="hello"></div>
</template>

<script>
    import {sample as OpenAPISampler} from 'openapi-sampler';

    export default {
        data() {
            return {
                spec: null,
            };
        },
        methods: {
            async fetchSpec() {
                const response = await fetch('https://cdn.rebilly.com/combined-api-specs/api-specs.json', {
                    method: 'GET',
                    headers: new Headers({
                        'content-type': 'application/json'
                    })
                });
                this.spec = await response.json();
                console.log('full spec', this.spec);
                console.log('\n\r\n\r');

                const mockResponse = this.mockAPI({
                    path: '/customers/5124845',
                    verb: 'get',
                    response: 200,
                }, {
                    overrides: {
                        customFields: this.mockResource('CustomField'),
                    }
                });

                console.log('mocked response body', mockResponse);
            },
            mockResource(schema, options) {
                if (!this.spec.components.schemas[schema]) {
                    throw new Error(`'schemas' [${schema}] not within OpenAPI Spec`);
                }
                return OpenAPISampler(this.spec.components.schemas[schema], options, this.spec);
            },
            mockAPI(call = {}, {options = {}, overrides = {}} = {}) {
                const {
                    path,
                    verb = 'get',
                    response,
                } = call;
                console.log(`mocking -> ${response} ${verb} ${path}`);

                let normalizedPath = path;
                const find = Object.keys(this.spec.paths).find((def) => {
                    if (def === normalizedPath) {
                        return true;
                    }
                    const pathRegx = new RegExp(`^${def.replace(/\{[a-zA-Z0-9-]+\}/, '[a-zA-Z0-9-]+').replace(/\//g, '\\/')}$`);
                    return pathRegx.test(normalizedPath);
                });
                if (find) {
                    normalizedPath = find;
                }

                if (!this.spec.paths[normalizedPath]) {
                    throw new Error(`'path' [${normalizedPath}] not within OpenAPI Spec`);
                }

                if (!this.spec.paths[normalizedPath][verb]) {
                    throw new Error(`verb '${verb}' not within OpenAPI Spec for path '${normalizedPath}'`);
                }

                if (this.spec.paths[normalizedPath][verb].responses[response]) {
                    let responseSpec = this.spec.paths[normalizedPath][verb].responses[response];
                    if (responseSpec.$ref) {
                        const referencePath = responseSpec.$ref.split('/');
                        responseSpec = referencePath.reduce((definition, particle) => {
                            if (particle === '#') {return definition}
                            return definition[particle];
                        }, this.spec);
                    }

                    let sample = OpenAPISampler(responseSpec.content['application/json'].schema, options, this.spec);

                    // TODO: Add support for overrideing elements within a collection
                    if (overrides) {
                        const responseKeys = Object.keys(sample);
                        const overrideKeys = Object.keys(overrides);
                        const isNotAddingKeys = overrideKeys.every((key) => responseKeys.includes(key));
                        if (isNotAddingKeys) {
                            return {
                                ...sample,
                                ...overrides,
                            }
                        }
                        throw new Error(`One or more keys '[${overrideKeys}]' in override do not have matches within OpenAPI Spec response for '${response} ${verb} ${normalizedPath}' '[${responseKeys}]'`)
                    }
                    return sample
                } else {
                    throw new Error(`response '${response}' not within OpenAPI spec for '${verb} ${normalizedPath}'`);
                }
            },
        },
        created() {
            this.fetchSpec();
        },
    }
</script>
