# GitOps Http Application Sample

## HTTP Application 
This Gitops sample provides a standard HTTP component consisting of a deployment, service and route. 

The following day 2 edit/update operations supported:
    set/get image - updates the image for this component 
    set/get replicas

{% if product.product_description %}
        <div class="pf-v5-c-form__field">
            <label class="pf-v5-c-form__label"">
                            <span class=" pf-v5-c-form__label-text">Product description</span>
            </label>
            <div id="product-description-view" class="pf-v5-c-form__field-control">
                {{ product.product_description | safe }}
            </div>
        </div>
        {% endif %}
