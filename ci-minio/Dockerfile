# use sha256 to prevent minio from changing behavior under `latest` tag
FROM minio/minio@sha256:14cea493d9a34af32f524e538b8346cf79f3321eff8e708c1e2960462bd8936e
# add CMD so that the image works in GH CI, see our ci yaml for more details
CMD ["server", "/data", "--address=0.0.0.0:9000"]
