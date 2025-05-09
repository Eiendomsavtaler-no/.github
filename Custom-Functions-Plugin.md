# Oversikt over Custom Functions Plugin for Eiendomsavtaler.no

Denne dokumentasjonen gir en strukturert oversikt over alle funksjoner og kortkoder som er definert i **Custom Functions Plugin** (versjon 23.01.2025). Pluginet er utviklet for å støtte Eiendomsavtaler.no sin digitale markedsplass, med fokus på diskrete off-market-transaksjoner, skreddersydde visninger av annonser og automatisering av kategorisering.

---

## 1. Sikkerhet

### 1.1. Forhindre direkte tilgang  
```php
if (!defined('ABSPATH')) {
    exit;
}
````

Sikrer at ingen kan kalle plugin-filen direkte i nettleseren.

---

## 2. Shortcodes for brukerdata

| Shortcode              | Funksjon                     | Beskrivelse                                                   |
| ---------------------- | ---------------------------- | ------------------------------------------------------------- |
| `[username]`           | `display_username_shortcode` | Viser innlogget brukers fornavn, eller “Guest”.               |
| `[profile-first-name]` | `display_first_name`         | Henter og viser ACF-feltet `first_name` for nåværende bruker. |
| `[profile-last-name]`  | `display_last_name`          | Henter og viser ACF-feltet `last_name`.                       |
| `[profile-selskap]`    | `display_selskap`            | Henter og viser ACF-feltet `text-1` (selskap).                |
| `[post_id]`            | `display_post_id`            | Viser ID-en til gjeldende innlegg.                            |
| `[current_page_url]`   | `get_current_page_url`       | Returnerer full URL til gjeldende side.                       |

Hver shortcode registreres via `add_shortcode()` og gir enkel tilgang til bruker- eller sideinformasjon i Elementor, Gutenberg eller tema­filer.

---

## 3. Filtrering og formatering

### 3.1. Fjerne forfatterinfo fra link-preview

```php
add_filter('oembed_response_data', 'disable_embeds_filter_oembed_response_data_');
function disable_embeds_filter_oembed_response_data_($data) {
    unset($data['author_url'], $data['author_name']);
    return $data;
}
```

Sletter `author_url` og `author_name` fra oEmbed-data, for diskret deling av eksterne lenker.

### 3.2. Formatere tall fra ACF

```php
add_filter('acf/format_value/type=number', 'format_acf_number', 20, 3);
function format_acf_number($value, $post_id, $field) {
    if (is_numeric($value)) {
        $disable = get_field('no_thousand_separator', $post_id);
        return $disable ? $value : number_format($value, 0, ',', ' ');
    }
    return $value;
}
```

Legger til tusenskilletegn på tallfelt, med mulighet for å deaktivere via ACF-feltet `no_thousand_separator`.

### 3.3. Yield-kalkulator

```php
add_shortcode('yield_calculator', 'display_yield');
function display_yield() {
    $leie = get_field('leieinntekter');
    $pris = get_field('prisestimat');
    if ($leie && $pris && $pris != 0) {
        return number_format(($leie / $pris) * 100, 2);
    }
    return 'Fyll ut leieinntekter og prisestimat riktig.';
}
```

Beregner og viser årlig yield (leieinntekter ÷ prisestimat × 100).

---

## 4. Tilgangskontroll

### 4.1. Blokkere wp-admin for ikke-administratorer

```php
add_action('admin_init', 'block_wp_admin_for_authors');
function block_wp_admin_for_authors() {
    if (is_admin() && !current_user_can('administrator') && !defined('DOING_AJAX')) {
        wp_redirect(home_url());
        exit;
    }
}
```

Hindrer at forfattere og abonnenter får tilgang til back-end.

### 4.2. Deaktivere utloggingsbekreftelse

```php
add_action('check_admin_referer', 'logout_without_confirm', 10, 2);
function logout_without_confirm($action, $result) {
    if ($action == 'log-out' && !isset($_GET['_wpnonce'])) {
        wp_redirect(wp_logout_url(home_url()));
        exit;
    }
}
```

Fjerner standardbekreftelse ved utlogging for en sømløs brukeropplevelse.

---

## 5. Dynamisk oppførsel ved innsending

### 5.1. Legge til query-parameter etter skjema

```php
add_action('init', 'add_query_param_after_submission');
function add_query_param_after_submission() {
    if (isset($_GET['form_submitted']) && $_GET['form_submitted'] === 'true') {
        add_action('wp_footer', 'trigger_elementor_popup');
    }
}
```

Trigger et Elementor-popup når `?form_submitted=true` i URL, f.eks. etter innsending av kontaktskjema.

---

## 6. Visning av annonser

### 6.1. Hente og vise «annonser»

```php
function get_annonser_posts() { /* … */ }
```

* Spørring (`WP_Query`) mot `post_type = 'annonser'`
* Viser inntil 8 kort med tittel, bilde, prisestimat, fylke og kategori
* Brukes som shortcode eller direkte PHP-kall for markedsplass-forside

---

## 7. Elementor-utvidelser

### 7.1. Egnet display-condition for «hovedkategori»

```php
add_filter('elementor/theme/settings/display_conditions', 'add_hovedkategori_to_elementor_conditions');
function add_hovedkategori_to_elementor_conditions($conditions) { /* … */ }
```

### 7.2. Tilpasse Loop Grid-query

```php
add_action('elementor/query/alle-annonser', 'custom_loop_grid_query');
function custom_loop_grid_query($query) { /* … */ }
```

Filtrerer annonser i Elementor-loop basert på URL-parameter `e-filter-…-hovedkategori`.

---

## 8. Godkjenning av annonser

### 8.1. Direkte godkjenning via shortcode

```php
add_shortcode('approve_post', 'approve_any_post_shortcode');
function approve_any_post_shortcode() { /* … */ }
```

Viser en knapp for å publisere utkast («Godkjenn annonse») direkte fra annonse­siden.

---

## 9. Automatisk kategorisering og bilder

### 9.1. Tilordne taksonomi og featured image ved innsending

```php
add_action('wpuf_add_post_after_insert',    'assign_acf_taxonomy_and_featured_image_based_on_form', 10, 2);
add_action('wpuf_draft_post_after_insert',  'assign_acf_taxonomy_and_featured_image_based_on_form', 10, 2);

function assign_acf_taxonomy_and_featured_image_based_on_form($post_id, $form_id) { /* … */ }
```

* Kartlegger skjema-ID til «hovedkategori»
* Setter standard eller spesifikt featured image
* Endrer post-status til utkast

### 9.2. Hjelpefunksjon for standardbilde

```php
function set_standard_featured_image($post_id, $image_url) { /* … */ }
```

Laster opp og setter et forhåndsdefinert bilde hvis ingen annen thumbnail finnes.

---

## 10. Placeholder

### 10.1. Favorittliste (ikke implementert)

```php
function ccc_my_favorite_list_custom_template($my_favorite_post_id) { }
```

Reserverer plass for egen mal til favorittinnlegg, kan utvides ved behov.
