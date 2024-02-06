# Utilisez une image Java JRE comme base
FROM openjdk:11-jre-slim

# Copiez le fichier .class dans l'image
COPY Main.class /app/Main.class

# Définir le répertoire de travail
WORKDIR /app

# Exécuter l'application Java
CMD ["java", "Main"]
