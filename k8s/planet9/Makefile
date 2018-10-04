include ../vendor/marketplace-tools/app.Makefile
include ../vendor/marketplace-tools/crd.Makefile
include ../vendor/marketplace-tools/gcloud.Makefile
include ../vendor/marketplace-tools/marketplace.Makefile
include ../vendor/marketplace-tools/ubbagent.Makefile
include ../vendor/marketplace-tools/var.Makefile


TAG ?= 1.0
$(info ---- TAG = $(TAG))

APP_DEPLOYER_IMAGE ?= $(REGISTRY)/planet9/deployer:$(TAG)
APP_INSTANCE_NAME ?= planet9-1

ifdef REPLICAS
  REPLICAS_FIELD = , "REPLICAS": "$(REPLICAS)"
endif

ifdef IMAGE_PLANET9
  IMAGE_PLANET9_FIELD = , "IMAGE_PLANET9": "$(IMAGE_PLANET9)"
endif

ifdef IMAGE_POSTGRESQL
  IMAGE_POSTGRESQL_FIELD = , "IMAGE_POSTGRESQL": "$(IMAGE_POSTGRESQL)"
endif

APP_PARAMETERS ?= { \
  "APP_INSTANCE_NAME": "$(APP_INSTANCE_NAME)", \
  "NAMESPACE": "$(NAMESPACE)" \
  $(REPLICAS_FIELD) \
  $(IMAGE_PLANET9_FIELD) \
}
APP_TEST_PARAMETERS ?= {}


app/build:: .build/planet9/deployer \
            .build/planet9/planet9 \
            .build/planet9/postgresql

.build/planet9: | .build
	mkdir -p "$@"

.build/planet9/deployer: deployer/* \
                               manifest/* \
                               schema.yaml \
                               .build/var/APP_DEPLOYER_IMAGE \
                               .build/var/REGISTRY \
                               .build/var/TAG \
                               | .build/planet9
	docker build \
	    --build-arg REGISTRY="$(REGISTRY)/planet9" \
	    --build-arg TAG="$(TAG)" \
	    --tag "$(APP_DEPLOYER_IMAGE)" \
	    -f deployer/Dockerfile \
	    .
	docker push "$(APP_DEPLOYER_IMAGE)"
	@touch "$@"


.build/planet9/planet9: .build/var/REGISTRY \
                        .build/var/TAG \
                        | .build/planet9
	docker pull gcr.io/planet9-k8s/planet9:$(TAG)
	docker tag gcr.io/planet9-k8s/planet9:$(TAG) \
	    "$(REGISTRY)/planet9:$(TAG)"
	docker push "$(REGISTRY)/planet9:$(TAG)"
	@touch "$@"

.build/planet9/postgresql: .build/var/REGISTRY \
                           .build/var/TAG \
                           | .build/planet9
	docker pull launcher.gcr.io/google/postgresql9:latest
	docker tag launcher.gcr.io/google/postgresql9:latest \
	    "$(REGISTRY)/planet9/postgresql:$(TAG)"
	docker push "$(REGISTRY)/planet9/postgresql:$(TAG)"
	@touch "$@"
