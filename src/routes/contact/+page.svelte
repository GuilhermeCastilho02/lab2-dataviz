<svelte:head>
  <title>Contact me</title>
</svelte:head>
<h1> Guilherme Castilho</h1>

<p> Contact</p>

<form action="mailto:foo@example.com" method="get">
    <label for="email">Email:<input id="email" name="email" type="email"><br></label><br>
    <label for="subject">Assunto:<input id="subject" name="subject" type="text"><br></label><br>
    
    <label for="body">Texto:<textarea id="body" name="body"></textarea><br></label><br>
    
    <button>Enviar</button><br>
</form>
