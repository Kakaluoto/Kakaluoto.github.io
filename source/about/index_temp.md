---
layout: false
---
{% raw %}
<!doctype html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Catch The Cat</title>
    <style>
        * {
            padding: 0;
            margin: 0;
        }

        body {
            background-color: #eeeeee;
        }
    
        #catch-the-cat {
            width: 100%;
            margin-top: 32px;
            text-align: center;
        }
    </style>
</head>
<body>
    <script src="phaser.min.js"></script>
    <script src="catch-the-cat.js"></script>
    <div id="catch-the-cat"></div>
    <!-- 1行对应43px -->
    <script>
      window.game = new CatchTheCatGame({
        w: 15,
        h: 15,
        r: 20,
        backgroundColor: 0xffffff,
        parent: 'catch-the-cat',
        statusBarAlign: 'center',
        credit: 'for kakaluoto\'s blog'
      });
    </script>
</body>
</html>

{% endraw %}