---
title: "Troubleshooting Valet: Making Hidden Sites For Invisible In Valet-Sites."
seoTitle: "Fixing Hidden Sites in Valet-Sites"
seoDescription: "Learn how to troubleshoot and display hidden sites in Valet to ensure they appear in your available sites list"
datePublished: Mon Jan 20 2025 05:21:31 GMT+0000 (Coordinated Universal Time)
cuid: cm64lmifz000g09jyhmezhw8w
slug: troubleshooting-valet-making-hidden-sites-for-invisible-in-valet-sites
tags: linux, valet

---

Valet sites display the folders you want to access. The default folder is www in your user folder (/home/username/www) also valet parks some folders but does not show in valet sites.

```xml
<!doctype html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, user-scalable=no, initial-scale=1.0, maximum-scale=1.0, minimum-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Valet Sites</title>
    <style>
        html,
        body {
            padding: 0;
            margin: 0;
            font-family: monospace, sans-serif;
            line-height: 1.45;
            font-size: 16px;
            letter-spacing: 0;
            position: relative;
        }

        ul {
            padding: 0;
            margin: 0;
            list-style: none;
            text-align: center;
        }

        ul>li {
            display: inline-block;
            margin: 10px 5px;
        }

        ul>li>a {
            text-decoration: none;
            padding: 15px 30px;
            display: block;
            text-transform: capitalize;
            color: #2d2d2d;
            text-align: center;
            border: 1px solid rgba(51, 51, 51, 0.25);
            box-shadow: 2px 2px 5px #d4d4d4;
            transition: all .15s linear;
        }

        ul>li>a:hover {
            box-shadow: 1px 1px 3px #d4d4d4;
        }

        ul>li>a:visited {
            color: #0042bd;
            border-color: #0042bd;
        }

        ul>li>a:before {
            content: "";
            height: 50px;
            width: 100%;
            display: inline-block;
            text-align: center;
            background: url('image path or base64');
            background-size: 50px 50px;
            background-repeat: no-repeat;
            background-position: center center;
        }

        h1 {
            text-align: center;
            margin: 15px 0 15px;
            border-bottom: 1px solid #ccc;
            line-height: 1.6;
            padding: 0px 0 10px;
        }

        @media (max-width: 767px) {
            h1 {
                font-size: 25px;
            }

            ul>li>a {
                padding: 5px 15px;
            }
        }

        @media (max-width: 479px) {
            h1 {
                font-size: 20px;
            }
        }
    </style>
</head>

<body>
    <h1>Valet Available Sites</h1>
    <?php if (isset($availableSites)) { ?>
        <ul>
            <?php
            foreach ($availableSites as $sitePath => $availableSite) {
                echo "<li><a href='/valet-sites?use={$sitePath}'>{$availableSite}</a></li>";
            }
            ?>
        </ul>
    <?php } ?>
</body>

</html>
```

So how to get available sites to show?

```php
<?php
    $directory = '/home/shazeedul/www'; // Replace with the path to your directory
    if (is_dir($directory)) {
        $files = scandir($directory);
        // filter only directories
        $sites          = array_filter($files, function ($file) use ($directory) {
            return is_dir($directory . '/' . $file) && !in_array($file, ['.', '..', '.git']);
        });
        $availableSites = [];
        foreach ($sites as $key => $site) {
            $key                  = strtolower(str_replace(' ', '-', $site));
            $availableSites[$key] = ucwords(str_replace('-', ' ', $site));
        }
    } else {
        $availableSites = [];
    }
?>
```