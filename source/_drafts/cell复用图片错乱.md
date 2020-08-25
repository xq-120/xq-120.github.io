```objc
@interface ZAMatchmakerListCollectionCell : UICollectionViewCell

@property (nonatomic, strong) NSIndexPath *indexPath;
@property (nonatomic, weak) UICollectionView *collectionView;

- (void)configCellWithModel:(ZANewLiveListBroadcastElement *)model delegate:(id)delegate;

@end


// MARK: UICollectionViewDataSource
- (UICollectionViewCell *)collectionView:(UICollectionView *)collectionView cellForItemAtIndexPath:(NSIndexPath *)indexPath
{
    ZAMatchmakerListCollectionCell *cell = [collectionView dequeueReusableCellWithReuseIdentifier:NSStringFromClass([ZAMatchmakerListCollectionCell class]) forIndexPath:indexPath];
    cell.indexPath = indexPath;
    cell.collectionView = collectionView;
    ZANewLiveListBroadcastElement *model = [self.presenter adapterForRowInSection:indexPath.section atIndex:indexPath.row];
    [cell configCellWithModel:model delegate:self];
    return cell;
}


- (void)prepareForReuse
{
    [super prepareForReuse];
    
    self.guestAvatarImgView.image = nil;
    self.guestInfoLabel.text = nil;
    self.matchmakerNicknameLabel.text = nil;
    self.matchmakerRoomNameLabel.text = nil;
    self.matchmakerAvatarImgView.image = nil;
}

// 配置
- (void)configCellWithModel:(ZANewLiveListBroadcastElement *)model delegate:(id)delegate {

    if (model.model.facilitateContent.length > 0) { //无异性嘉宾
        //取默认头像
        self.guestAvatarImgView.image = [UIImage imageNamed:@"matchmaker_default_icon"];
        [self.guestNicknameBadgeView setNickname:model.model.facilitateName badgeImgs:nil];
        self.guestInfoLabel.text = model.model.facilitateContent;
    } else { //有异性嘉宾
        ZALiveUserModel *oppositeGuestModel = [self getOppositeSexUserModelWithLiveElement:model];
        if (oppositeGuestModel) { //取到异性嘉宾
            __weak typeof(self) weakSelf = self;
            NSURL *guestAvataUrl = [NSURL URLWithURLString:oppositeGuestModel.avatarURL size:CGSizeMake(kGuestAvatarWH, kGuestAvatarWH)];
            [self.guestAvatarImgView sd_setImageWithURL:guestAvataUrl placeholderImage:[ZAAppThemeManager avatarPlaceHolderImageWithGender:ZAGender_Null] options:SDWebImageAvoidAutoSetImage completed:^(UIImage * _Nullable image, NSError * _Nullable error, SDImageCacheType cacheType, NSURL * _Nullable imageURL) {
                if (!error && image) {
                    NSIndexPath *currentIndexPath = [weakSelf.collectionView indexPathForCell:weakSelf];
                    if (currentIndexPath && currentIndexPath.row != weakSelf.indexPath.row) {
                        return ;
                    }
                    
                    weakSelf.guestAvatarImgView.image = image;
                }
            }];
            [self.guestNicknameBadgeView setNickname:oppositeGuestModel.nickname badgeImgs:[self getFlagImageWithModel:oppositeGuestModel]];
            self.guestInfoLabel.text = [NSString stringWithFormat:@"%@ | %@岁", oppositeGuestModel.workCityString, @(oppositeGuestModel.age)];
        } else { //没取到:服务器BUG.设置为默认图像,防止UI异常
            self.guestAvatarImgView.image = [UIImage imageNamed:@"matchmaker_default_icon"];
            self.guestInfoLabel.text = @"";
        }
    }
    
    ZALiveUserModel *anchor = [self getAnchorModelWithLiveElement:model];
    NSURL *matchmakerAvatarUrl = [NSURL URLWithURLString:anchor.avatarURL size:CGSizeMake(kMatchmakerAvatarWH, kMatchmakerAvatarWH)];
    [self.matchmakerAvatarImgView sd_setImageWithURL:matchmakerAvatarUrl placeholderImage:[ZAAppThemeManager avatarPlaceHolderImageWithGender:ZAGender_Null] completed:nil];
    self.matchmakerNicknameLabel.text = anchor.nickname;
    self.matchmakerRoomNameLabel.text = model.model.liveTitle;
}

```

