# WordPress Output Buffering in Plugins and Themes

https://core.trac.wordpress.org/ticket/43258

## Probably the best way to use full page output buffering in WordPress plugins and themes
$priority: Lower numbers correspond with earlier execution, so your output buffer will contain the content of higher value buffers. 

```php
$priority = 10;
add_action('template_redirect', function(){
	ob_start('callback_function');
}, $priority);

add_action('shutdown', function(){
	ob_end_flush();
}, -1 * $priority);

function callback_function($content_of_the_buffer){
	//Do whatever you need to do with the content of the buffer
	
	return $content_of_the_buffer;
}
```
Example priorities to maintain the right ouput buffer order:
- $priority = -100; // Full page cache plugin
- $priority = -50; // JS minifier plugin
- $priority = -50; // CSS minifier plugin
- $priority = 100; // Contact form embedder plugin

## Output Buffer Tester - WordPress Plugin

This plugin helps you to find the place where your output buffer was closed by other plugin or theme.

https://wordpress.org/plugins/output-buffer-tester/


## Plugins

#### Smart Slider 3 https://wordpress.org/plugins/smart-slider-3/

```php
add_action('template_redirect', 'N2Wordpress::outputStart', 10000);

add_action('shutdown', 'N2Wordpress::closeOutputBuffers', -10000);
add_action('pp_end_html', 'N2Wordpress::closeOutputBuffers', -10000); // ProPhoto 6 theme
add_action('headway_html_close', 'N2Wordpress::closeOutputBuffers', -10000);
```

```php
public static function outputStart() {
    static $started = false;
    if(!$started) {
	    $started = true;

	    ob_start( "N2Wordpress::platformRenderEnd" );
    }
}

public static function closeOutputBuffers(){
    $handlers = ob_list_handlers();
    if(in_array('N2Wordpress::platformRenderEnd', $handlers)){
	    for($i = count($handlers)-1; $i >= 0; $i--){
		    if($handlers[$i] === 'N2Wordpress::platformRenderEnd'){
			self::finalizeCssJs();
		    }
		    ob_end_flush();
		    if($handlers[$i] === 'N2Wordpress::platformRenderEnd'){
			    break;
		    }
	    }
    }
}
```

#### JCH Optimize https://wordpress.org/plugins/jch-optimize/

Note: On shutdown action, JCH optimize closes all ouput buffer.

```php
add_action('init', 'jch_buffer_start', 0);
add_action('template_redirect', 'jch_buffer_start', 0);
add_action('shutdown', 'jch_buffer_end', -1);

function jch_buffer_start()
{
        ob_start();
}

function jch_buffer_end()
{
        while ($level = ob_get_level())
        {
                if (JchOptimizeHelper::validateHtml($sHtml = ob_get_contents()))
                {
                        $sOptimizedHtml = jchoptimize($sHtml);

                        ob_clean();

                        echo $sOptimizedHtml;

                        break;
                }

                ob_end_flush();

                //buffer not flushed for some reason.
                if ($level == ob_get_level())
                {
                        break;
                }
        }
}
```

#### Autoptimize https://wordpress.org/plugins/autoptimize/
Autoptimize hooks into template_redirect by default, but you can change that with constants.

```php
// Hook to wordpress
if (defined('AUTOPTIMIZE_INIT_EARLIER')) {
    add_action('init','autoptimize_start_buffering',-1);
} else {
    if (!defined('AUTOPTIMIZE_HOOK_INTO')) { define('AUTOPTIMIZE_HOOK_INTO', 'template_redirect'); }
    add_action(constant("AUTOPTIMIZE_HOOK_INTO"),'autoptimize_start_buffering',2);
}
```

#### WP Rocket https://wp-rocket.me
Starts the output buffer in advanced-cache.php
```php
ob_start( 'do_rocket_callback' );
```

#### Yoast SEO https://wordpress.org/plugins/wordpress-seo/

```php
// For WordPress functions below 4.4.
if ( $this->options['forcerewritetitle'] === true && ! current_theme_supports( 'title-tag' ) ) {
	add_action( 'template_redirect', array( $this, 'force_rewrite_output_buffer' ), 99999 );
	add_action( 'wp_footer', array( $this, 'flush_cache' ), - 1 );
}
```

#### W3 Total Cache https://wordpress.org/plugins/w3-total-cache/

It starts output buffering in the plugin's main thread with callaback.

```php
if ( $this->can_ob() ) {
	ob_start( array(
			$this,
			'ob_callback'
		) );
}
```

#### Speed Contact Bar https://wordpress.org/plugins/speed-contact-bar/
Note: I have contacted with the developer to change the template_include filter to template_redirect action: https://wordpress.org/support/topic/smart-slider-3-conflict-and-code-improvement-suggestion/

```php
add_filter( 'template_include', array( $this, 'activate_buffer' ), 1 );
add_filter( 'shutdown', array( $this, 'include_contact_bar' ), 0 );
```

```php
public function activate_buffer( $template ) {
	// activate output buffer
	ob_start();
	// return html without changes
	return $template;
}

public function include_contact_bar() {
...
	// get current buffer content and clean buffer
	$content = ob_get_clean(); 
			
```

#### Ultimate Reviews https://wordpress.org/plugins/ultimate-reviews/
Improper usage of output buffer as it conflicts with others. I just the code and this output buffer does nothing. https://wordpress.org/support/topic/improper-usage-of-output-buffers/

```php
function EWD_URP_add_ob_start() {
    ob_start();
}
add_action('init', 'EWD_URP_add_ob_start');

function EWD_URP_flush_ob_end() {
    ob_end_flush();
}
add_action('wp_footer', 'EWD_URP_flush_ob_end');
```

## Themes

#### ProPhoto 6 https://www.prophoto.com/

It starts output buffering in the themes's functions.php thread with callback.

Closes the output buffer on the ```pp_end_html``` action

```PHP
protected function capture($key)
{
ob_start();
$this->on('pp_end_html', array($this, 'endCapture'));
}
```

#### YooTheme Warp 7 https://yootheme.com/themes/warp-framework

Opens 2 output budder in the header.php and closes both in footer.php

```PHP
// start output buffer to capture wp_head
ob_start();

wp_enqueue_script('jquery');
wp_head();

// start output buffer to capture content for use in footer.php
ob_start();
```

```PHP
// get content from output buffer and set a slot for the template renderer
$warp['template']->set('content', ob_get_clean());
$warp['template']->set('wp_head', ob_get_clean());
```
