Karafka uses [Celluloid](https://celluloid.io/) actors to handle listening to incoming connections. Since each topic and group requires a separate connection (which means that we have a connection per controller) we do this concurrently. It means, that for each route, you will have one additional thread running.