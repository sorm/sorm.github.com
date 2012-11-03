For debugging purposes it can be useful to see the SQL that SORM emits under the hood. For that you need to set its logging level to "debug". To do that you just need to add the following VM option to your app execution:

    -Dorg.slf4j.simplelogger.log.sorm=debug

Here's a complete sample execution command:

    java -Dorg.slf4j.simplelogger.log.sorm=debug -jar MyAmazingApp.jar
