default: test

include ../../../helpers/examples.mk

RELEASE := helm-es-security
ELASTICSEARCH_IMAGE := docker.elastic.co/elasticsearch/elasticsearch:$(STACK_VERSION)

install:
	helm upgrade --wait --timeout=600s --install --values ./security.yml $(RELEASE) ../../

purge:
	kubectl delete secrets elastic-credentials elastic-certificates elastic-certificate-pem elastic-certificate-ca-pem elastic-certificate-cert-pem elastic-certificate-key-pem elastic-tls-certificates || true
	helm del $(RELEASE)

test: secrets install goss

pull-elasticsearch-image:
	docker pull $(ELASTICSEARCH_IMAGE)

secrets:
	docker rm -f elastic-helm-charts-certs || true
	rm -f elastic-certificates.p12 elastic-certificate.pem elastic-stack-ca.p12 elastic-certificate-ca.pem elastic-certificate-cert.pem elastic-certificate-key.pem elastic-tls-certificates.crt elastic-tls-certificates.key || true
	password=$$([ ! -z "$$ELASTIC_PASSWORD" ] && echo $$ELASTIC_PASSWORD || echo $$(docker run --rm busybox:1.31.1 /bin/sh -c "< /dev/urandom tr -cd '[:alnum:]' | head -c20")) && \
	docker run --name elastic-helm-charts-certs -i -w /app \
		$(ELASTICSEARCH_IMAGE) \
		/bin/sh -c " \
			elasticsearch-certutil ca --out /app/elastic-stack-ca.p12 --pass '' && \
			elasticsearch-certutil cert --name security-master --dns es.hello.local,kibana.hello.local --ca /app/elastic-stack-ca.p12 --pass '' --ca-pass '' --out /app/elastic-certificates.p12" && \
	docker cp elastic-helm-charts-certs:/app/elastic-certificates.p12 ./ && \
	docker rm -f elastic-helm-charts-certs && \
	openssl pkcs12 -nodes -passin pass:'' -in elastic-certificates.p12 -out elastic-certificate.pem && \
	openssl pkcs12 -nodes -passin pass:'' -in elastic-certificates.p12 -out elastic-certificate-ca.pem -cacerts -nokeys && \
	openssl pkcs12 -nodes -passin pass:'' -in elastic-certificates.p12 -out elastic-certificate-cert.pem -clcerts -nokeys && \
	openssl pkcs12 -nodes -passin pass:'' -in elastic-certificates.p12 -out elastic-certificate-key.pem -nocerts && \
	kubectl create secret generic elastic-certificate-ca-pem --from-file=elastic-certificate-ca.pem && \
	kubectl create secret generic elastic-certificate-cert-pem --from-file=elastic-certificate-cert.pem && \
	kubectl create secret generic elastic-certificate-key-pem --from-file=elastic-certificate-key.pem && \
	kubectl create secret generic elastic-certificates --from-file=elastic-certificates.p12 && \
	kubectl create secret generic elastic-certificate-pem --from-file=elastic-certificate.pem && \
	kubectl create secret generic elastic-credentials  --from-literal=password=$$password --from-literal=username=elastic && \
	openssl crl2pkcs7 -nocrl -certfile elastic-certificate-cert.pem | openssl pkcs7 -print_certs -out elastic-tls-certificates.crt && \
	openssl pkey -in elastic-certificate-key.pem -out elastic-tls-certificates.key && \
	kubectl create secret tls elastic-tls-certificates --cert=elastic-tls-certificates.crt --key=elastic-tls-certificates.key