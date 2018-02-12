# WordPress Ouput Buffering in Plugins and Themes


## Plugins

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

## Themes

### ProPhoto 6 https://www.prophoto.com/

It starts output buffering in the themes's functions.php thread with callaback.

Closes the output buffer on the ```pp_end_html``` action

```PHP
protected function capture($key)
{
ob_start();
$this->on('pp_end_html', array($this, 'endCapture'));
}
```
