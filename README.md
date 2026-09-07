# List all named volumes
container volume ls

# Remove specific named volumes
container volume rm app_node_modules_bpr npm_global_cache

# List all images
container image ls

# Remove the specific development image
container image rm blog-post-react

# System prune (Removes all stopped containers, unused networks, and dangling images)
container system prune

# System prune including all unused volumes
container system prune --volumes
