# WordPress Ouput Buffering in Plugins and Themes


## Plugins

### Smart Slider 3 https://wordpress.org/plugins/smart-slider-3/

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

### JCH Optimize https://wordpress.org/plugins/jch-optimize/

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

### Yoast SEO https://wordpress.org/plugins/wordpress-seo/

```php
// For WordPress functions below 4.4.
if ( $this->options['forcerewritetitle'] === true && ! current_theme_supports( 'title-tag' ) ) {
	add_action( 'template_redirect', array( $this, 'force_rewrite_output_buffer' ), 99999 );
	add_action( 'wp_footer', array( $this, 'flush_cache' ), - 1 );
}
```

### W3 Total Cache https://wordpress.org/plugins/w3-total-cache/

It starts output buffering in the plugin's main thread with callaback.

```php
if ( $this->can_ob() ) {
	ob_start( array(
			$this,
			'ob_callback'
		) );
}
```

### Speed Contact Bar https://wordpress.org/plugins/speed-contact-bar/
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

## Themes

### ProPhoto 6 https://www.prophoto.com/

It starts output buffering in the themes's functions.php thread with callback.

Closes the output buffer on the ```pp_end_html``` action

```PHP
protected function capture($key)
{
ob_start();
$this->on('pp_end_html', array($this, 'endCapture'));
}
```

### YooTheme Warp 7 https://yootheme.com/themes/warp-framework

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
