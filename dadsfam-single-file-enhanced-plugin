<?php
/**
 * Plugin Name: DadsFam Enhanced Weight & Distance Shipping
 * Description: Flexible weight-based shipping with optional real-time distance calculation using Google Routes API. Base rate up to a set weight within a free distance range, plus per-kg and per-km charges beyond thresholds.
 * Author: DadsFam
 * Version: 2.2.1
 * Requires PHP: 7.4
 * License: GPLv2 or later
 */

if ( ! defined( 'ABSPATH' ) ) {
    exit;
}

// Ensure WooCommerce is active
add_action( 'plugins_loaded', function () {
    if ( class_exists( 'WC_Shipping_Method' ) ) {
        class DadsFam_Enhanced_Single_Shipping extends WC_Shipping_Method {
            public function __construct( $instance_id = 0 ) {
                $this->id                 = 'dadsfam_enhanced_single';
                $this->instance_id        = absint( $instance_id );
                $this->method_title       = __( 'DadsFam Enhanced Shipping', 'dadsfam-enhanced-single' );
                $this->method_description = __( 'Weight-based shipping with optional distance-based surcharges using Google Routes API.', 'dadsfam-enhanced-single' );
                $this->supports           = array(
                    'shipping-zones',
                    'instance-settings'
                );

                $this->init_form_fields();
                $this->init_settings();

                // Load settings
                $this->enabled                = $this->get_option( 'enabled', 'yes' );
                $this->title                  = $this->get_option( 'title', __( 'DadsFam Shipping', 'dadsfam-enhanced-single' ) );
                $this->currency_label         = get_woocommerce_currency_symbol();

                // Weight settings
                $this->threshold_kg           = floatval( $this->get_option( 'threshold_kg', 15 ) );
                $this->min_price              = floatval( $this->get_option( 'min_price', 120 ) );
                $this->rate_per_kg            = floatval( $this->get_option( 'rate_per_kg', 4 ) );
                $this->calc_mode              = $this->get_option( 'calc_mode', 'over_threshold_plus_base' );
                $this->round_weight           = $this->get_option( 'round_weight', 'none' );
                $this->min_price_applies_when = $this->get_option( 'min_price_applies_when', 'lte_threshold' );

                // Distance settings
                $this->enable_distance        = $this->get_option( 'enable_distance', 'yes' );
                $this->google_api_key         = $this->get_option( 'google_api_key', '' );
                $this->store_address          = $this->get_option( 'store_address', '' );
                $this->free_distance_range    = floatval( $this->get_option( 'free_distance_range', 25 ) );
                $this->max_delivery_distance  = floatval( $this->get_option( 'max_delivery_distance', 100 ) );
                $this->distance_unit          = $this->get_option( 'distance_unit', 'km' );
                $this->distance_rate          = floatval( $this->get_option( 'distance_rate', 3 ) );

                add_action( 'woocommerce_update_options_shipping_' . $this->id, array( $this, 'process_admin_options' ) );

                // Force script loading for theme compatibility
                add_action( 'wp_footer', array( $this, 'force_enqueue_scripts' ) );
            }

            public function init_form_fields() {
                $this->instance_form_fields = array(
                    // Basic settings
                    'basic_title' => array(
                        'title' => __( 'Basic Settings', 'dadsfam-enhanced-single' ),
                                           'type'  => 'title',
                                           'description' => __( 'Enable and configure the shipping method', 'dadsfam-enhanced-single' ),
                    ),
                    'enabled' => array(
                        'title'   => __( 'Enable', 'dadsfam-enhanced-single' ),
                                       'type'    => 'checkbox',
                                       'label'   => __( 'Enable this shipping method', 'dadsfam-enhanced-single' ),
                                       'default' => 'yes',
                    ),
                    'title' => array(
                        'title'   => __( 'Method Title', 'dadsfam-enhanced-single' ),
                                     'type'    => 'text',
                                     'default' => __( 'DadsFam Shipping', 'dadsfam-enhanced-single' ),
                                     'description' => __( 'This controls the title customers see during checkout.', 'dadsfam-enhanced-single' ),
                    ),

                    // Weight-based pricing
                    'weight_title' => array(
                        'title' => __( 'Weight-Based Pricing', 'dadsfam-enhanced-single' ),
                                            'type'  => 'title',
                                            'description' => __( 'Configure weight-based pricing structure', 'dadsfam-enhanced-single' ),
                    ),
                    'threshold_kg' => array(
                        'title'       => __( 'Weight Threshold (kg)', 'dadsfam-enhanced-single' ),
                                            'type'        => 'number',
                                            'description' => __( 'Weight included in the base price (default: 15kg).', 'dadsfam-enhanced-single' ),
                                            'default'     => 15,
                                            'custom_attributes' => array( 'step' => '0.1', 'min' => '0' ),
                    ),
                    'min_price' => array(
                        'title'       => __( 'Base Price', 'dadsfam-enhanced-single' ),
                                         'type'        => 'number',
                                         'description' => __( 'Base delivery price when weight is within threshold.', 'dadsfam-enhanced-single' ),
                                         'default'     => 120,
                                         'custom_attributes' => array( 'step' => '1', 'min' => '0' ),
                    ),
                    'rate_per_kg' => array(
                        'title'       => __( 'Rate per kg Above Threshold', 'dadsfam-enhanced-single' ),
                                           'type'        => 'number',
                                           'description' => __( 'Additional charge per kg above the weight threshold.', 'dadsfam-enhanced-single' ),
                                           'default'     => 4,
                                           'custom_attributes' => array( 'step' => '0.1', 'min' => '0' ),
                    ),
                    'calc_mode' => array(
                        'title'   => __( 'Weight Calculation Mode', 'dadsfam-enhanced-single' ),
                                         'type'    => 'select',
                                         'default' => 'over_threshold_plus_base',
                                         'options' => array(
                                             'over_threshold_plus_base' => __( 'Base price + extra per kg above threshold', 'dadsfam-enhanced-single' ),
                                                            'full_weight_no_base'      => __( 'Per kg rate for full weight when over threshold', 'dadsfam-enhanced-single' ),
                                         ),
                    ),
                    'round_weight' => array(
                        'title'   => __( 'Weight Rounding', 'dadsfam-enhanced-single' ),
                                            'type'    => 'select',
                                            'default' => 'none',
                                            'options' => array(
                                                'none'  => __( 'No rounding', 'dadsfam-enhanced-single' ),
                                                               'ceil'  => __( 'Round up to next kg', 'dadsfam-enhanced-single' ),
                                                               'floor' => __( 'Round down to whole kg', 'dadsfam-enhanced-single' ),
                                            ),
                    ),
                    'min_price_applies_when' => array(
                        'title'   => __( 'Minimum Price Application', 'dadsfam-enhanced-single' ),
                                                      'type'    => 'select',
                                                      'default' => 'lte_threshold',
                                                      'options' => array(
                                                          'lte_threshold'      => __( 'Use base price only when weight ≤ threshold', 'dadsfam-enhanced-single' ),
                                                                         'always_min_if_less' => __( 'Always apply base price as minimum', 'dadsfam-enhanced-single' ),
                                                      ),
                    ),

                    // Distance-based pricing
                    'distance_title' => array(
                        'title' => __( 'Distance-Based Pricing', 'dadsfam-enhanced-single' ),
                                              'type'  => 'title',
                                              'description' => __( 'Optional real-time distance calculation and surcharges', 'dadsfam-enhanced-single' ),
                    ),
                    'enable_distance' => array(
                        'title'   => __( 'Enable Distance Calculation', 'dadsfam-enhanced-single' ),
                                               'type'    => 'checkbox',
                                               'label'   => __( 'Calculate delivery distance and add charges beyond free range', 'dadsfam-enhanced-single' ),
                                               'default' => 'yes',
                    ),
                    'google_api_key' => array(
                        'title'       => __( 'Google Maps Routes API Key', 'dadsfam-enhanced-single' ),
                                              'type'        => 'text',
                                              'description' => __( 'Required for distance calculation. Get one from Google Cloud Console (Routes API must be enabled).', 'dadsfam-enhanced-single' ),
                                              'default'     => '',
                    ),
                    'store_address' => array(
                        'title'       => __( 'Store / Origin Address', 'dadsfam-enhanced-single' ),
                                             'type'        => 'textarea',
                                             'description' => __( 'Full address from where shipments originate. Used for distance calculation.', 'dadsfam-enhanced-single' ),
                                             'default'     => '',
                                             'placeholder' => '123 Example Street, City, Province, Country',
                    ),
                    'free_distance_range' => array(
                        'title'       => __( 'Free Distance Range (km)', 'dadsfam-enhanced-single' ),
                                                   'type'        => 'number',
                                                   'description' => __( 'Distance included in the base price with no extra charge (default: 25km).', 'dadsfam-enhanced-single' ),
                                                   'default'     => 25,
                                                   'custom_attributes' => array( 'step' => '1', 'min' => '1' ),
                    ),
                    'distance_rate' => array(
                        'title'       => __( 'Distance Rate Beyond Free Range', 'dadsfam-enhanced-single' ),
                                             'type'        => 'number',
                                             'description' => __( 'Additional charge per km (or mile) beyond the free distance range.', 'dadsfam-enhanced-single' ),
                                             'default'     => 3,
                                             'custom_attributes' => array( 'step' => '0.1', 'min' => '0' ),
                    ),
                    'max_delivery_distance' => array(
                        'title'       => __( 'Maximum Delivery Distance', 'dadsfam-enhanced-single' ),
                                                     'type'        => 'number',
                                                     'description' => __( 'Maximum distance you are willing to deliver. Shipping option will be hidden beyond this distance.', 'dadsfam-enhanced-single' ),
                                                     'default'     => 100,
                                                     'custom_attributes' => array( 'step' => '1', 'min' => '1' ),
                    ),
                    'distance_unit' => array(
                        'title'   => __( 'Distance Unit', 'dadsfam-enhanced-single' ),
                                             'type'    => 'select',
                                             'default' => 'km',
                                             'options' => array(
                                                 'km'    => __( 'Kilometers (km)', 'dadsfam-enhanced-single' ),
                                                                'miles' => __( 'Miles', 'dadsfam-enhanced-single' ),
                                             ),
                    ),
                );
            }

            public function force_enqueue_scripts() {
                if ( ! is_checkout() ) {
                    return;
                }

                if ( $this->enabled !== 'yes' || $this->enable_distance !== 'yes' || empty( $this->google_api_key ) ) {
                    return;
                }

                if ( ! wp_script_is( 'jquery', 'done' ) ) {
                    wp_enqueue_script( 'jquery' );
                }
                ?>
                <style>
                #dadsfam-info {
                margin: 10px 0 !important;
                padding: 12px 15px !important;
                border-radius: 4px !important;
                font-size: 14px !important;
                line-height: 1.4 !important;
                display: block !important;
                }
                .dadsfam-info { background: #f0f8ff !important; border-left: 4px solid #007cba !important; color: #333 !important; }
                .dadsfam-success { background: #eafaf1 !important; border-left: 4px solid #00a32a !important; color: #1e4620 !important; }
                .dadsfam-warning { background: #fff3cd !important; border-left: 4px solid #ffc107 !important; color: #856404 !important; }
                .dadsfam-error { background: #ffeaea !important; border-left: 4px solid #dc3232 !important; color: #721c24 !important; }
                .dadsfam-loading { background: #f8f9fa !important; border-left: 4px solid #007cba !important; color: #333 !important; }
                .dadsfam-spinner { display: inline-block !important; animation: dadsfam-spin 1s linear infinite !important; margin-right: 5px !important; }
                @keyframes dadsfam-spin { 0% { transform: rotate(0deg); } 100% { transform: rotate(360deg); } }
                </style>

                <script type="text/javascript">
                window.dadsfam_shipping = {
                    ajax_url: '<?php echo admin_url( 'admin-ajax.php' ); ?>',
            nonce: '<?php echo wp_create_nonce( 'dadsfam_distance_nonce' ); ?>',
            api_key: '<?php echo esc_js( $this->google_api_key ); ?>',
            store_address: '<?php echo esc_js( $this->store_address ); ?>',
            max_distance: <?php echo floatval( $this->max_delivery_distance ); ?>,
            free_range: <?php echo floatval( $this->free_distance_range ); ?>,
            distance_unit: '<?php echo esc_js( $this->distance_unit ); ?>'
                };

                (function() {
                    function initDadsFamCalculator() {
                        if (typeof jQuery === 'undefined') {
                            setTimeout(initDadsFamCalculator, 100);
                            return;
                        }

                        jQuery(document).ready(function($) {
                            var dadsfamCalc = {
                                debounceTimer: null,

                                init: function() {
                                    this.bindEvents();
                                    setTimeout(() => this.calculateDistance(), 3000);
                                },

                                bindEvents: function() {
                                    var self = this;

                                    $(document).on('change keyup blur', 'input[name*="address"], input[name*="city"], input[name*="postcode"], select[name*="country"], select[name*="state"]', function() {
                                        self.debounceCalculate();
                                    });

                                    $(document).on('change', '#ship-to-different-address-checkbox', function() {
                                        setTimeout(() => self.calculateDistance(), 500);
                                    });

                                    $(document).on('updated_checkout', function() {
                                        setTimeout(() => self.calculateDistance(), 1000);
                                    });
                                },

                                debounceCalculate: function() {
                                    clearTimeout(this.debounceTimer);
                                    this.debounceTimer = setTimeout(() => this.calculateDistance(), 2000);
                                },

                                getAddress: function() {
                                    var useShipping = $('#ship-to-different-address-checkbox').is(':checked');
                                    var prefix = useShipping ? 'shipping_' : 'billing_';
                        var parts = [];

                        var fieldPatterns = [
                            ['address_1', 'address_2', 'city', 'state', 'postcode', 'country'],
                            ['address-1', 'address-2', 'city', 'state', 'postcode', 'country'],
                            ['street_address', 'address_line_2', 'city', 'state', 'postal_code', 'country']
                        ];

                        for (var p = 0; p < fieldPatterns.length; p++) {
                            var pattern = fieldPatterns[p];
                            var tempParts = [];

                            for (var i = 0; i < pattern.length; i++) {
                                var field = pattern[i];
                                var selectors = [
                                    'input[name="' + prefix + field + '"]',
                                    'select[name="' + prefix + field + '"]',
                                    'textarea[name="' + prefix + field + '"]',
                                    'input[name="' + field + '"]',
                                    'select[name="' + field + '"]',
                                    '#' + prefix + field,
                                    '#' + field
                                ];

                                var val = '';
                        for (var s = 0; s < selectors.length; s++) {
                            var $field = $(selectors[s]);
                            if ($field.length && $field.val()) {
                                val = $field.val().trim();
                                break;
                            }
                        }

                        if (val) tempParts.push(val);
                            }

                            if (tempParts.length >= 2) {
                                parts = tempParts;
                                break;
                            }
                        }

                        if (parts.length === 0) {
                            $('input[type="text"], select, textarea').each(function() {
                                var name = ($(this).attr('name') || $(this).attr('id') || '').toLowerCase();
                                var val = $(this).val().trim();
                                if (val && (name.includes('address') || name.includes('city') ||
                                    name.includes('postcode') || name.includes('postal') ||
                                    name.includes('country') || name.includes('state'))) {
                                    parts.push(val);
                                    }
                            });
                        }

                        return parts.join(', ');
                                },

                                calculateDistance: function() {
                                    var self = this;
                                    var address = this.getAddress();

                                    if (!address || address.length < 10) {
                                        this.hideInfo();
                                        return;
                                    }

                                    this.showLoading();

                                    $.ajax({
                                        url: dadsfam_shipping.ajax_url,
                                        type: 'POST',
                                        data: {
                                            action: 'dadsfam_calculate_distance',
                                            nonce: dadsfam_shipping.nonce,
                                            address: address,
                                            store_address: dadsfam_shipping.store_address,
                                            api_key: dadsfam_shipping.api_key
                                        },
                                        success: function(response) {
                                            self.hideLoading();
                                            try {
                                                var data = JSON.parse(response);
                                                if (data.success) {
                                                    self.showDistanceInfo(data.data);
                                                    $(document.body).trigger('update_checkout');
                                                } else {
                                                    self.showError(data.error || 'Distance calculation failed');
                                                }
                                            } catch (e) {
                                                self.showError('Invalid server response');
                                            }
                                        },
                                        error: function(xhr, status, error) {
                                            self.hideLoading();
                                            self.showError('Network error: ' + error);
                                        }
                                    });
                                },

                                showLoading: function() {
                                    this.hideInfo();
                                    var html = '<div id="dadsfam-info" class="dadsfam-loading">' +
                        '<span class="dadsfam-spinner">⟳</span> Calculating delivery distance...</div>';

                        var targets = ['.woocommerce-shipping-methods', '#shipping_method', '.woocommerce-checkout-review-order-table'];
                        var inserted = false;

                        for (var i = 0; i < targets.length; i++) {
                            var $target = $(targets[i]);
                            if ($target.length) {
                                $target.after(html);
                                inserted = true;
                                break;
                            }
                        }

                        if (!inserted) $('body').append(html);
                                },

                                showDistanceInfo: function(data) {
                                    this.hideInfo();
                                    var distance = parseFloat(data.distance_km);
                                    var unit = dadsfam_shipping.distance_unit;
                                    var freeRange = dadsfam_shipping.free_range;
                                    var maxRange = dadsfam_shipping.max_distance;

                                    if (unit === 'miles') distance *= 0.621371;

                                    var distanceText = distance.toFixed(1) + ' ' + unit;
                                    var className = 'dadsfam-info';
                        var icon = '📍';
                        var message = 'Delivery distance: ' + distanceText;

                        if (distance > maxRange) {
                            className = 'dadsfam-error';
                        icon = '❌';
                        message = 'Sorry, we do not deliver beyond ' + maxRange + unit + '. Distance: ' + distanceText;
                        } else if (distance > freeRange) {
                            className = 'dadsfam-warning';
                        icon = '⚠️';
                        var extra = (distance - freeRange).toFixed(1);
                        message += ' (+R' + (extra * <?php echo floatval( $this->distance_rate ); ?>) + ' for ' + extra + unit + ' beyond free range)';
                        } else {
                            className = 'dadsfam-success';
                        icon = '✅';
                        message += ' (Within free delivery range)';
                        }

                        if (data.duration_text) message += ' • Est. ' + data.duration_text;

                        var html = '<div id="dadsfam-info" class="' + className + '">' + icon + ' ' + message + '</div>';

                                    var targets = ['.woocommerce-shipping-methods', '#shipping_method', '.woocommerce-checkout-review-order-table'];
                                    var inserted = false;

                                    for (var i = 0; i < targets.length; i++) {
                                        var $target = $(targets[i]);
                                        if ($target.length) {
                                            $target.after(html);
                                            inserted = true;
                                            break;
                                        }
                                    }

                                    if (!inserted) $('body').append(html);
                                },

                                showError: function(error) {
                                    this.hideInfo();
                                    var html = '<div id="dadsfam-info" class="dadsfam-error">⚠️ ' + error + '</div>';
                        $('.woocommerce-shipping-methods, #shipping_method').first().after(html);
                                },

                                hideInfo: function() {
                                    $('#dadsfam-info').remove();
                                },

                                hideLoading: function() {
                                    $('#dadsfam-info').remove();
                                }
                            };

                            dadsfamCalc.init();

                            $(document.body).on('updated_checkout updated_wc_div', function() {
                                setTimeout(() => dadsfamCalc.calculateDistance(), 1000);
                            });
                        });
                    }

                    initDadsFamCalculator();
                })();
                </script>
                <?php
            }

            public function calculate_shipping( $package = array() ) {
                // Weight calculation
                $weight = 0.0;
                foreach ( $package['contents'] as $item ) {
                    if ( $item['data'] && $item['quantity'] > 0 ) {
                        $product_weight = floatval( $item['data']->get_weight() );
                        $weight += ( $product_weight * $item['quantity'] );
                    }
                }

                if ( $this->round_weight === 'ceil' ) {
                    $weight = ceil( $weight );
                } elseif ( $this->round_weight === 'floor' ) {
                    $weight = floor( $weight );
                }

                $weight_cost = 0.0;
                if ( $weight <= $this->threshold_kg ) {
                    $weight_cost = $this->min_price;
                } else {
                    if ( $this->calc_mode === 'over_threshold_plus_base' ) {
                        $excess = max( 0, $weight - $this->threshold_kg );
                        $weight_cost = $this->min_price + ( $excess * $this->rate_per_kg );
                    } else {
                        $weight_cost = $weight * $this->rate_per_kg;
                        if ( $this->min_price_applies_when === 'always_min_if_less' && $weight_cost < $this->min_price ) {
                            $weight_cost = $this->min_price;
                        }
                    }
                }

                // Distance calculation
                $distance_cost = 0.0;
                $total_cost = $weight_cost;
                $delivery_info = '';

                if ( $this->enable_distance === 'yes' && !empty( $this->google_api_key ) && !empty( $this->store_address ) ) {
                    $customer_address = $this->get_customer_address( $package );

                    if ( !empty( $customer_address ) ) {
                        $distance_data = $this->calculate_distance_routes_api( $this->store_address, $customer_address );

                        if ( $distance_data && isset( $distance_data['distance_km'] ) ) {
                            $distance = $distance_data['distance_km'];

                            if ( $this->distance_unit === 'miles' ) {
                                $distance = $distance * 0.621371;
                            }

                            if ( $distance > $this->max_delivery_distance ) {
                                return; // Do not offer shipping beyond max distance
                            }

                            if ( $distance > $this->free_distance_range ) {
                                $chargeable_distance = $distance - $this->free_distance_range;
                                $distance_cost = $chargeable_distance * $this->distance_rate;
                            }

                            $total_cost = $weight_cost + $distance_cost;

                            $delivery_info = sprintf( ' (%.1f %s)', round( $distance, 1 ), $this->distance_unit );
                        }
                    }
                }

                $this->add_rate( array(
                    'id'    => $this->id . ':' . $this->instance_id,
                    'label' => $this->title . $delivery_info,
                    'cost'  => round( $total_cost, 2 ),
                                       'meta_data' => array(
                                           'weight_cost'   => round( $weight_cost, 2 ),
                                                            'distance_cost' => round( $distance_cost, 2 ),
                                                            'total_weight'  => $weight,
                                       ),
                                       'package' => $package,
                ) );
            }

            private function get_customer_address( $package ) {
                $address_parts = array();

                if ( !empty( $package['destination']['address'] ) ) $address_parts[] = $package['destination']['address'];
                if ( !empty( $package['destination']['address_2'] ) ) $address_parts[] = $package['destination']['address_2'];
                if ( !empty( $package['destination']['city'] ) ) $address_parts[] = $package['destination']['city'];
                if ( !empty( $package['destination']['state'] ) ) $address_parts[] = $package['destination']['state'];
                if ( !empty( $package['destination']['postcode'] ) ) $address_parts[] = $package['destination']['postcode'];
                if ( !empty( $package['destination']['country'] ) ) $address_parts[] = $package['destination']['country'];

                return implode( ', ', array_filter( $address_parts ) );
            }

            private function calculate_distance_routes_api( $origin, $destination ) {
                if ( empty( $this->google_api_key ) ) {
                    return false;
                }

                $cache_key = 'dadsfam_routes_' . md5( $origin . $destination );
                $cached_result = get_transient( $cache_key );

                if ( $cached_result !== false ) {
                    return $cached_result;
                }

                $url = 'https://routes.googleapis.com/directions/v2:computeRoutes';

                $request_body = array(
                    'origin' => array( 'address' => $origin ),
                                      'destination' => array( 'address' => $destination ),
                                      'travelMode' => 'DRIVE',
                                      'routingPreference' => 'TRAFFIC_UNAWARE',
                                      'computeAlternativeRoutes' => false,
                                      'languageCode' => 'en-US',
                                      'units' => 'METRIC'
                );

                $response = wp_remote_post( $url, array(
                    'timeout' => 30,
                    'headers' => array(
                        'Content-Type' => 'application/json',
                        'X-Goog-Api-Key' => $this->google_api_key,
                        'X-Goog-FieldMask' => 'routes.duration,routes.distanceMeters',
                        'Referer' => home_url(),
                                       'User-Agent' => 'DadsFam-WordPress-Plugin/2.2.1'
                    ),
                    'body' => json_encode( $request_body )
                ) );

                if ( is_wp_error( $response ) ) {
                    error_log( 'DadsFam Routes API Error: ' . $response->get_error_message() );
                    return false;
                }

                $response_code = wp_remote_retrieve_response_code( $response );
                $body = wp_remote_retrieve_body( $response );

                if ( $response_code !== 200 ) {
                    error_log( 'DadsFam Routes API HTTP Error ' . $response_code . ': ' . $body );
                    return false;
                }

                $data = json_decode( $body, true );

                if ( isset( $data['routes'] ) && !empty( $data['routes'] ) ) {
                    $route = $data['routes'][0];

                    if ( isset( $route['distanceMeters'] ) ) {
                        $distance_km = $route['distanceMeters'] / 1000;
                        $duration_seconds = isset( $route['duration'] ) ? intval( str_replace( 's', '', $route['duration'] ) ) : 0;
                        $duration_text = $this->format_duration( $duration_seconds );

                        $result = array(
                            'distance_km'   => $distance_km,
                            'distance_text' => number_format( $distance_km, 1 ) . ' km',
                                        'duration_text' => $duration_text,
                        );

                        set_transient( $cache_key, $result, HOUR_IN_SECONDS );
                        return $result;
                    }
                }

                return false;
            }

            private function format_duration( $seconds ) {
                if ( $seconds < 60 ) return $seconds . ' sec';
                if ( $seconds < 3600 ) return ceil( $seconds / 60 ) . ' min';

                $hours = floor( $seconds / 3600 );
                $minutes = ceil( ( $seconds % 3600 ) / 60 );
                return $hours . 'h ' . $minutes . 'm';
            }
        }

        // Register shipping method
        add_filter( 'woocommerce_shipping_methods', function ( $methods ) {
            $methods['dadsfam_enhanced_single'] = 'DadsFam_Enhanced_Single_Shipping';
            return $methods;
        } );

        // AJAX handler for frontend distance calculation
        add_action( 'wp_ajax_dadsfam_calculate_distance', 'dadsfam_ajax_calculate_distance' );
        add_action( 'wp_ajax_nopriv_dadsfam_calculate_distance', 'dadsfam_ajax_calculate_distance' );

        function dadsfam_ajax_calculate_distance() {
            check_ajax_referer( 'dadsfam_distance_nonce', 'nonce' );

            $customer_address = sanitize_text_field( $_POST['address'] ?? '' );
            $store_address    = sanitize_text_field( $_POST['store_address'] ?? '' );
            $api_key          = sanitize_text_field( $_POST['api_key'] ?? '' );

            if ( empty( $customer_address ) || empty( $store_address ) || empty( $api_key ) ) {
                wp_send_json_error( 'Missing required parameters' );
            }

            $cache_key = 'dadsfam_routes_' . md5( $store_address . $customer_address );
            $cached = get_transient( $cache_key );

            if ( $cached !== false ) {
                wp_send_json_success( $cached );
            }

            // Reuse the same Routes API logic
            $shipping = new DadsFam_Enhanced_Single_Shipping();
            $result = $shipping->calculate_distance_routes_api( $store_address, $customer_address );

            if ( $result ) {
                wp_send_json_success( $result );
            } else {
                wp_send_json_error( 'Could not calculate distance' );
            }
        }
    }
} );
