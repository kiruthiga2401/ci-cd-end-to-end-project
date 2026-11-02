# புதிய மற்றும் சரியான Java 11 Base Image
FROM eclipse-temurin:11-jre-alpine

# உங்கள் Maven build உருவாக்கிய உண்மையான Jar ஃபைலின் பெயரை இங்கே கொடுக்கவும்
ARG artifact=target/*.jar

WORKDIR /opt/app

COPY ${artifact} app.jar

ENTRYPOINT ["java","-jar","app.jar"]
